# Phone Verification Login with SMS OTP Evidence (When Reports Are Attached)

TL;DR: Let the backend own the SMS OTP resend deadline, attempt budget, and one-time transition; let the browser display that deadline, never create it. For an edtech workflow that emails a generated student report as an attachment, keep three claims separate: the phone challenge succeeded, the report was released, and the email was accepted for delivery. None proves that a person read the attachment.

That separation is the decision rule. A countdown is useful interface feedback, but compliance evidence must come from durable server-side state and append-only events. The critical invariant is narrow: one challenge can authorize one report-release operation, once, within its validity window.

The timer lies.

## What evidence should a phone verification login retain after SMS OTP?

A browser timer proves almost nothing. A learner can refresh, edit local storage, open two tabs, or call the endpoint without rendering the page. The backend must return an absolute `resend_at` timestamp and reject an early resend even when the client presents an enabled button. The client may calculate `max(0, resend_at - now)` every second; clock skew can make that display slightly wrong, while the server decision remains authoritative.

The evidence boundary matters more than the widget. SMS verification supports the claim that someone with access to a phone number completed a challenge. It does not establish identity, guardianship, or consent by itself. NIST SP 800-63B treats use of the public switched telephone network as a restricted authenticator and calls for consideration of risks such as SIM change and number porting. High-impact actions therefore need a stronger policy than “the code matched.”

Six decimal digits provide one million possible values, which is why code secrecy is not enough: short expiry, attempt limits, rate limits, and replay prevention carry much of the security burden. A six-digit value is a format choice, not evidence of a six-digit level of assurance.

Email creates a second boundary. DKIM associates a signed message with a domain and protects signed fields against modification in transit; it does not prove inbox placement or human access. Open tracking is weaker still because Mail Privacy Protection can download remote content in the background. For a student report, record provider acceptance and later delivery events as transport observations, not as proof that the recipient opened or understood the document.

| Option | Authoritative state | Useful evidence | Failure boundary | Valid fit |
|---|---|---|---|---|
| Browser-only countdown | Browser memory or storage | UI activity | Refresh, tampering, parallel tabs | Cosmetic throttling only |
| Server challenge row | Transactional datastore | Issue, resend, verify, expiry decisions | Datastore availability and clock discipline | Most report-release flows |
| Stateless signed challenge | Signed client token | Token issuance and validation | Replay control still needs server state | Low-risk flows where replay has no consequence |
| Queue-backed challenge and release | Datastore plus durable queue | Decision and dispatch lifecycle | Duplicate delivery and delayed workers | Attachments generated asynchronously |

**Choose the server challenge row when an auditor must reconstruct why access was granted.** Add a durable queue when report generation or mail submission can outlive the request. Stateless tokens look attractive, but once the system needs resend limits, attempt counters, revocation, and exactly-once consumption, state has returned through another door.

## Invariants and failure boundaries

Store a random challenge identifier, a keyed digest of the code, the normalized phone number or a separately protected reference, `expires_at`, `resend_at`, attempts remaining, send count, status, and a version used for conditional updates. Do not store the plaintext OTP. Separate that mutable row from an append-only audit stream containing event type, challenge ID, report ID, policy version, timestamp, and a request correlation ID. Keep message bodies, OTPs, and report contents out of the audit stream.

Retention is a policy decision, not an implementation default. GDPR Article 5 requires purpose limitation, data minimization, accuracy, storage limitation, and appropriate security. That argues for explicit retention periods per record class: OTP material can expire quickly, operational message metadata may live longer for investigation, and formal compliance evidence may have a different schedule. The exact periods depend on jurisdiction, institutional obligations, and legal review; there is no defensible universal number.

Name the failures before choosing the schema. Duplicate send-code requests can race. An SMS submission can time out after the downstream service accepted it. Two correct verification requests can arrive together. A report-generation worker can finish after authorization expires. An email worker can retry after a crash and attach the same report twice. A recycled phone number can belong to a different person months later. Each case needs either a transactional state transition, an idempotency key, or an explicit policy decision.

Consider the least dramatic race, because it is easy to miss in review. A parent taps resend as the browser timer reaches zero, a second tab does the same, and both requests read the old row before either writes. If each request generates a different code and the SMS jobs leave in the opposite order, the code displayed as newest in server state may arrive first, followed by an already invalid code. A conditional update accepts one transition; the conflicting request returns the stored `resend_at` rather than generating another message. The audit trail should record one accepted transition and one rejected early or stale transition, not pretend that both sends were independent successes. This is also why a last-code policy is insufficient unless code rotation and outbox insertion share a transaction.

The hard rule is short: **verification and consumption must be atomic**. Marking a challenge verified in one transaction and releasing the report in another creates a replay window. Use a conditional update on status and version, then create the release command in the same transaction through an outbox record. Workers should claim that command idempotently; “exactly once” is an application invariant built over retries, not a promise to infer from a successful HTTP response.

## Critical path in Python

The service below leaves transport and persistence behind narrow interfaces. Repository methods are expected to use transactions and row-level or compare-and-swap protection. `resend_at` is returned to the web application as an absolute UTC timestamp; disabling the button in the browser remains a presentation concern.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import hmac
import secrets


@dataclass(frozen=True)
class OtpPolicy:
    ttl: timedelta
    resend_delay: timedelta
    max_attempts: int


def utc_now() -> datetime:
    return datetime.now(timezone.utc)


def digest(challenge_id: str, code: str, secret: bytes) -> str:
    payload = f"{challenge_id}:{code}".encode("utf-8")
    return hmac.new(secret, payload, hashlib.sha256).hexdigest()


def verify_and_release(repo, mail_outbox, audit, challenge_id: str,
                       submitted_code: str, secret: bytes) -> bool:
    now = utc_now()
    with repo.transaction():
        challenge = repo.lock_challenge(challenge_id)
        if challenge["status"] != "pending" or now >= challenge["expires_at"]:
            return False
        if challenge["attempts_left"] <= 0:
            return False

        expected = challenge["digest"]
        actual = digest(challenge_id, submitted_code, secret)
        if not hmac.compare_digest(expected, actual):
            repo.decrement_attempts(
                challenge_id, expected_version=challenge["version"]
            )
            audit.append(
                "otp_rejected", challenge_id, challenge["report_id"], now
            )
            return False

        repo.consume_challenge(
            challenge_id, expected_version=challenge["version"]
        )
        mail_outbox.enqueue_report(
            challenge["report_id"],
            dedupe_key=f"report:{challenge['report_id']}:verified-release",
        )
        audit.append(
            "report_release_authorized",
            challenge_id,
            challenge["report_id"],
            now,
        )
        return True
```

A resend method follows the same pattern: lock the row, require `pending`, require `now >= resend_at`, reject expired or send-exhausted challenges, rotate the code digest, increment both version and send count, advance `resend_at`, and enqueue the SMS inside the transaction. Returning the existing deadline for an early request is preferable to trusting a client-supplied countdown. Apply request-level rate limits by account, phone reference, and network risk signals as separate controls; a per-row cooldown alone does not stop creation of many rows.

Do not log `submitted_code`. Do not put it in an analytics event. Those two mistakes convert a short-lived secret into a replicated data-retention problem.

Logs spread.

## Operations, tests, and review evidence

Test the state machine under concurrency, not only through happy-path route tests. Two simultaneous verifies must produce one consumed transition and one report outbox item. Two resends at the boundary must rotate the digest once. A worker retry with the same dedupe key must not submit a second attachment. Time should be injectable in tests so expiry and cooldown cases do not sleep or depend on a laptop clock.

Deployment needs boring controls: UTC timestamps, synchronized hosts, encrypted datastore fields where the threat model requires them, managed signing keys with rotation, and migrations that tolerate old workers during a rolling release. Metrics should count issued, rejected, expired, rate-limited, consumed, and delivery-state events without using phone numbers as labels. Alert on ratios and queue age, while remembering that a delivery receipt describes a carrier or mailbox event rather than the recipient's intent.

For an audit sample, reconstruct one report release from immutable identifiers: policy version, challenge creation, resend decisions, successful consumption, outbox creation, report artifact checksum, mail submission, and final transport status. Access to that trail should itself be logged. If the reconstruction requires joining on plaintext phone numbers or reading the attachment, the evidence model is collecting too much.

The report artifact deserves storage discipline too. Generate it once, assign a stable ID and checksum, scan it before release, restrict object access to the mail worker, and delete it under the applicable retention policy. Avoid embedding a long-lived public object URL in the message. The attachment and its audit metadata have different access patterns and should not share a bucket policy merely because both are report data.

## Why reject a stateless countdown token?

A signed token can carry an expiry and a resend timestamp, and signature verification can detect alteration. It remains a reasonable option for a low-risk notification where replay causes no protected action and where the system deliberately accepts weak revocation. It is a poor ledger for this case. Attempt exhaustion, code rotation, phone-risk changes, report withdrawal, and single-use release all need a current decision that every application instance observes consistently.

That does not mean every component needs strong consistency. The challenge transition and release outbox do. Analytics, dashboards, and delivery summaries can lag, provided they never authorize access. This split keeps the transactional core small while making its durability claim testable.

Keep that boundary explicit.

**The countdown is never the control.** The defensible design is a backend-owned challenge state machine, an atomic handoff to report delivery, and evidence labels that stop at what each system actually observed.

## References

- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
