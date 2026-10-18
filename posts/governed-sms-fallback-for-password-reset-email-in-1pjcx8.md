# Governed SMS Fallback for Password Reset Email in Edtech Support

Choose email as the normal password-reset channel, and add SMS only as an explicitly enrolled recovery path whose identity risk, consent record, retention, and support ownership are acceptable. **TL;DR:** an email delivery delay must not automatically trigger a text message. For an edtech contact form, route a learner's recovery request by durable state: self-service email first, a separately verified SMS option only when it was established before the incident, and manual support for every ambiguous case. This costs more integration effort than an indiscriminate fallback, but it prevents a delivery optimization from quietly becoming an account-takeover path.

The decision rests on three invariants. A reset capability is single-use and short-lived; a channel change never proves identity by itself; and every transition can be reconstructed without storing the reset secret. Those invariants matter more than the messaging transport. Email and SMS both fail, sometimes after accepting a message, so an API success can record dispatch acceptance but cannot establish that the learner received anything.

## Should a password reset email strategy include SMS fallback?

Start by separating the recovery state machine from the contact-form queue. A learner may submit "I cannot sign in" several times, an instructor may report the same issue for a class, and a provider callback may arrive after the user has already recovered. These are events, not independent authorization decisions. The recovery service owns token issuance and consumption; the router owns categorization and assignment; transport adapters own attempts. No transport gets to mint a second token merely because the first attempt looks slow.

Four failure boundaries follow. First, normalize the public response so the form does not reveal whether an email address belongs to a learner. OWASP's Forgot Password guidance recommends a consistent message and response time for existent and nonexistent accounts, rate limiting, random single-use expiring tokens, and no account change until a valid token is presented. Second, deduplicate retries against a stable recovery request identifier. Third, treat provider acceptance, delivery, and token redemption as distinct facts. Last, redact destinations in routine logs; an address or phone number is operational data, not a convenient correlation key.

Silence is a state.

A practical record can contain `request_id`, a keyed digest of the normalized account lookup, `channel`, `attempt_no`, `accepted_at`, `delivery_event`, `expires_at`, and `consumed_at`. Keep the token itself out of that record. Store only a verifier suitable for checking the presented secret, and make consumption an atomic conditional update so concurrent clicks cannot both win. The exact expiration is a local risk decision; it should be documented and tested rather than copied from a messaging example.

The contact-form router should classify recovery language into a restricted queue and attach only the opaque request ID. Support staff need status such as `email_dispatched`, `sms_eligible`, or `manual_review`; they do not need a reset URL, token verifier, full phone number, or free-form copy of sensitive classroom context. This is where data-layer discipline earns its keep: append attempts for audit, but enforce the one-time transition at the authoritative recovery row.

Retries lie.

## Comparing the channel policies

The cheapest practical design is the smallest policy the team can operate correctly, not the transport with the lowest advertised unit charge. Pricing changes, while identity boundaries persist. Integration effort includes enrollment, consent capture where applicable, destination changes, webhook authentication, idempotency, rate limits, deletion workflows, accessibility, on-call diagnosis, and manual recovery.

| Policy | Integration effort | Useful boundary | Principal failure modes |
|---|---:|---|---|
| Email only | Low | Users have a maintained mailbox and support can handle exceptions | Spam filtering, delayed mail, compromised mailbox, inaccessible school domain |
| Email plus pre-enrolled SMS | High | A verified number already exists and the risk model accepts a second possession channel | Number recycling, SIM-swap exposure, shared family phones, roaming or carrier delay, stale consent |
| User-selected email or SMS at request time | Medium in code, high in security review | Rarely justified without prior binding of both destinations | Account enumeration, attacker-directed channel switching, inconsistent throttling |
| Email plus manual review | Medium operationally | Small populations or high-risk exceptions | Queue backlog, social engineering, inconsistent agent decisions, excess data collection |

For a school system, shared devices and guardian contact details make the data model especially awkward. A phone number collected for attendance notices is not automatically a learner-controlled recovery factor. GDPR requires a lawful basis and data minimization for personal-data processing, while its security principle requires measures appropriate to risk; those are design questions for counsel and the controller, not boxes that a messaging SDK can check. In the United States, the FTC describes CAN-SPAM as applying to commercial email and distinguishes transactional or relationship content. A password-reset message should stay narrowly transactional: do not turn the recovery template into a promotional newsletter and assume the classification is unchanged.

This is not legal advice, and geography alone is an unreliable policy switch. The relevant facts include the user's relationship to the institution, age and guardian model, why a phone number was collected, and which entity controls the data. Record the policy version evaluated for each request so a later audit does not depend on reconstructing mutable configuration.

This policy has a real limitation: email-first recovery is not suitable when the institution cannot establish that learners control durable mailboxes, while pre-enrolled SMS is a poor fit when numbers are routinely shared or recycled. The trade-off is deliberate. A program with neither dependable channel needs an identity-proofing and manual-support design specific to its enrollment records; adding another transport does not close that gap. Conversely, a program with managed institutional mailboxes and trained support may rationally stay email-only because the extra SMS enrollment, deletion, monitoring, and incident paths would increase complexity without resolving its dominant failure mode.

That boundary is deliberate.

## The critical path as a state transition

The following Python uses generic ports and deliberately omits transport SDK calls. Its important property is ordering: eligibility is decided before dispatch, every send has an idempotency key, and an email timeout moves the request to review rather than automatically selecting SMS. The values are illustrative configuration, not statutory limits.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Protocol


class Channel(str, Enum):
    EMAIL = "email"
    SMS = "sms"


@dataclass(frozen=True)
class RecoveryPolicy:
    token_ttl: timedelta = timedelta(minutes=15)
    max_attempts: int = 3
    allow_pre_enrolled_sms: bool = False


@dataclass(frozen=True)
class RecoveryRequest:
    request_id: str
    account_ref: str
    email_ref: str
    sms_ref: str | None
    sms_enrolled_at: datetime | None
    attempts: int


class Store(Protocol):
    def issue_once(
        self, request_id: str, account_ref: str, expires_at: datetime
    ) -> str: ...

    def record_attempt(
        self, request_id: str, channel: Channel, idempotency_key: str
    ) -> None: ...


class Transport(Protocol):
    def send(
        self, destination_ref: str, message: str, idempotency_key: str
    ) -> None: ...


def dispatch_recovery(
    request: RecoveryRequest,
    policy: RecoveryPolicy,
    store: Store,
    email: Transport,
    sms: Transport,
    requested_channel: Channel = Channel.EMAIL,
) -> str:
    if request.attempts >= policy.max_attempts:
        return "manual_review"

    sms_is_eligible = (
        policy.allow_pre_enrolled_sms
        and request.sms_ref is not None
        and request.sms_enrolled_at is not None
    )
    if requested_channel is Channel.SMS and not sms_is_eligible:
        return "manual_review"

    now = datetime.now(timezone.utc)
    secret = store.issue_once(
        request.request_id, request.account_ref, now + policy.token_ttl
    )
    channel = requested_channel
    destination = request.email_ref if channel is Channel.EMAIL else request.sms_ref
    idempotency_key = f"{request.request_id}:{channel.value}:{request.attempts + 1}"
    message = f"Complete account recovery with this one-time secret: {secret}"

    store.record_attempt(request.request_id, channel, idempotency_key)
    transport = email if channel is Channel.EMAIL else sms
    transport.send(destination, message, idempotency_key)
    return f"{channel.value}_dispatched"
```

A production implementation must avoid returning or logging the secret after message construction, authenticate asynchronous delivery events, and compare callback transitions against the current request state. It also needs one atomic consume operation, not a read followed by a write. If two browser tabs present the same token, exactly one transition from active to consumed should succeed.

Template rendering belongs behind the same boundary. Mustache's specification describes variables as HTML-escaped by default and triple mustaches or ampersands as unescaped; that behavior helps with HTML email but does not make arbitrary reset URLs safe in every context. Build the URL from a fixed origin and an encoded token, pass it as data, and keep the SMS template plain. Snapshot-test both rendered forms with hostile display names, missing optional fields, and the longest supported localization.

Test transitions, not just messages. Exercise duplicate form submissions, transport timeouts followed by late acceptance, callbacks delivered twice and out of order, simultaneous token consumption, expired tokens, a changed email address, a recycled phone number reported by support, and deletion of contact data while audit metadata remains under its documented retention rule. Metrics should count transitions by coarse reason and channel without placing destinations or secrets in labels. Alerts belong on sustained queue age, dispatch error ratios, and manual-review growth, not on a single late callback.

## Why automatic SMS fallback was rejected

The rejected design starts a timer after sending email and texts the same recovery link if no delivery event arrives. It looks compact. It is also built on a false inference: absence of a delivery event does not prove failed delivery, and delivery does not prove that the intended person controls the endpoint. The timer creates two live copies of a bearer capability, expands personal-data processing, and turns transport telemetry into an authentication decision.

**Do not fail open across channels.** If email cannot be used, present the same neutral public response and route an opaque case to manual review. Offer SMS only when the number was verified and bound under a documented recovery policy before the current request; even then, issue a fresh single-use capability and invalidate the prior one atomically instead of duplicating it.

Automatic fallback does have a valid use case for low-risk notifications that confer no authority: a class cancellation, for example, may be sent over a second enrolled channel when the first is unavailable. That logic should live in a notification policy, not be reused for password recovery. The distinction is capability, not message length.

The final architecture decision is therefore email-first self-service, optional pre-enrolled SMS by explicit policy, and a manual queue as the failure sink. Review the decision whenever the enrollment model, population, jurisdictional analysis, or support capacity changes. Until then, the router remains intentionally boring: it classifies, deduplicates, and points staff to state; it never improvises identity proof.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://mustache.github.io/mustache.5.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.rfc-editor.org/rfc/rfc5321
