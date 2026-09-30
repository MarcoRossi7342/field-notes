# Node.js Signup Event Ledger — 4 User Channel Email SMS Suppression Records

Send an e-commerce signup verification link only after a policy service has resolved the user's channel preference and checked a channel-specific suppression record; then preserve the decision, policy version, and delivery attempt as separate append-only evidence. **The deciding constraint is not delivery speed but the ability to prove what the system knew when it sent.** A Node.js API may accept the signup, but it should not let an email or SMS SDK decide eligibility inside the request handler.

Short answer: model four durable records—verification intent, preference version, suppression state, and dispatch decision—and join them using opaque identifiers. Keep raw contact details out of event payloads where possible. A later opt-out must stop a queued promotional send, while a narrowly defined transactional verification message must follow the organization's documented policy and applicable law; never infer that exception from a provider's successful API response.

## How should a Node.js event notification system enforce user channel preferences?

A delivery receipt proves that a downstream system accepted or processed a message. It does not prove consent, the absence of an opt-out, or why the chosen channel was permitted. Those claims require records controlled by the application.

That distinction matters.

For each signup, store an immutable verification intent with an intent ID, user ID, purpose, creation time, expiration time, and a digest of the verification token rather than the token itself. Store preferences as versioned facts, not a mutable pair of booleans. A preference change should create a new version containing its effective time, source, and policy basis. Suppressions need their own history because an address or phone number may become ineligible independently of a user's preference—for example, after an email hard bounce or an SMS stop request.

The fourth record is the dispatch decision. It names the exact preference and suppression versions examined, the policy revision applied, the selected channel, and a reason code. The provider attempt belongs in a related record because retries can produce several attempts for one decision. This separation is fussy on purpose: overwriting `status = sent` destroys the sequence an auditor needs.

Do not put the verification link in logs. Store a token digest and an expiry, and make redemption atomic so two concurrent requests cannot both consume it. Thirty-minute expiry may be a reasonable product choice, but it is not a universal compliance rule; document whichever interval the risk owner approves.

## Invariants and failure boundaries

The design has five invariants. A dispatcher reads one committed preference version. It evaluates suppression immediately before each attempt, not only when the job is enqueued. It never changes purpose from signup verification to marketing. It records the decision before crossing the delivery boundary. And every retry reuses the same intent while receiving a distinct attempt ID.

These rules expose the uncomfortable failures instead of hiding them. A queue can delay work until after an opt-out. Two workers can claim the same job. A provider can accept a request and time out before returning an identifier, leaving the caller uncertain whether a message exists. Webhooks can arrive late, twice, or out of order. A recycled phone number can make an old account preference unsafe evidence for a new holder. None of those cases is fixed by an SDK retry flag.

Consider one precise race. At 10:00:00 the signup transaction writes an intent and queues work; at 10:00:02 the user sends an SMS opt-out; at 10:00:03 the suppression consumer commits it; and at 10:00:05 a worker wakes up. A preference snapshot captured at enqueue time permits the message even though the current suppression record denies it. Checking again at dispatch closes that gap, but introduces another one: if the suppression store is unavailable, delivery must pause rather than silently treat “unknown” as “allowed.” The limitation is latency and availability coupling. The trade-off is deliberate because an unprovable send is harder to repair than a delayed verification prompt, while a product with a stricter signup latency objective may instead need a replicated policy read model with measured staleness bounds.

Retries are dangerous here.

Use uniqueness constraints for `intent_id + channel + decision_version`, and treat a timeout after submission as an unknown outcome rather than an automatic invitation to send again. Provider callbacks update attempt state idempotently by stable external identifiers; they do not rewrite the original eligibility decision. Suppression ingestion is monotonic for the same source event, while an explicit re-consent creates a new fact instead of deleting history.

| Dispatch model | Evidence quality | Principal failure boundary | Appropriate use |
|---|---|---|---|
| Inline send from signup request | Weak: policy input and attempt are easily collapsed | Client retries and ambiguous provider timeouts | Low-risk prototypes with no regulated evidence requirement |
| Queue with preference snapshot | Better, but the snapshot can age before dispatch | Opt-out races during queue delay | Messages whose eligibility cannot change while queued |
| Queue with dispatch-time policy check | Strong when inputs and decision are versioned | Policy store availability becomes part of the critical path | Verification flows requiring reproducible decisions |
| Transactional outbox plus policy check | Strongest recovery story in this comparison | Database retention and dispatcher operations require discipline | Systems that must reconcile intents, decisions, and attempts |

The final row costs more operationally. An outbox table needs retention, indexes, worker leases, and replay tooling; keeping it forever also creates a data-minimization problem. Evidence retention should therefore have a written schedule, and contact data should be tokenized or hashed with a keyed construction appropriate to the lookup threat model. An ordinary unsalted hash of a phone number is readily enumerable.

This design is not appropriate for every system. Its explicit history increases storage, deletion-workflow, and schema-migration costs, and the policy check adds a dependency to the dispatch path. A synthetic internal test harness doesn't need that burden. A regulated or disputed real-recipient flow often does, provided the team can operate the evidence store and define retention rather than keeping everything indefinitely.

## Critical path: decide, record, then deliver

The following Python expresses the boundary even when the surrounding service is Node.js: the language of the transport adapter is less important than the transaction contract. Repository methods represent parameterized database operations, and the adapter receives a short-lived link only after eligibility is committed.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class Channel(str, Enum):
    EMAIL = "email"
    SMS = "sms"


@dataclass(frozen=True)
class DispatchRequest:
    intent_id: str
    subject_id: str
    purpose: str
    candidate_channels: tuple[Channel, ...]


def decide_and_record(request, repositories, policy):
    now = datetime.now(timezone.utc)
    with repositories.transaction() as tx:
        intent = tx.intents.lock(request.intent_id)
        if intent.consumed_at or intent.expires_at <= now:
            return None

        preference = tx.preferences.effective_at(request.subject_id, now)
        for channel in request.candidate_channels:
            destination = tx.destinations.active(request.subject_id, channel)
            if destination is None:
                continue

            suppression = tx.suppressions.current(destination.lookup_key, channel)
            result = policy.evaluate(
                purpose=request.purpose,
                channel=channel,
                preference=preference,
                suppression=suppression,
            )
            if not result.allowed:
                continue

            return tx.decisions.insert_once(
                intent_id=request.intent_id,
                channel=channel,
                destination_id=destination.id,
                preference_version=preference.version,
                suppression_version=suppression.version if suppression else None,
                policy_version=policy.version,
                reason_code=result.reason_code,
                decided_at=now,
            )

        return tx.decisions.record_no_eligible_channel(
            intent_id=request.intent_id,
            policy_version=policy.version,
            decided_at=now,
        )
```

A worker claims the recorded decision, rechecks any policy input that can change under the organization's rules, creates an attempt row, and calls the channel adapter with an idempotency key if that interface supports one. The database remains the system of record for why the attempt was allowed. Provider events supply transport evidence such as accepted, delivered, bounced, or failed, but their vocabulary should be mapped into an internal state machine without erasing the original payload reference.

This is also where observability must avoid becoming a second contact database. Metrics can count decisions by channel and reason code. Traces can carry intent and attempt IDs. Logs should exclude addresses, phone numbers, verification tokens, and full provider callback bodies; retain protected raw evidence only where a defined investigation process requires it.

## Testing the races that happy paths miss

A unit test for “email preferred” proves very little. The valuable tests place state changes between queueing, deciding, sending, and receiving callbacks.

Run at least these cases: an SMS stop event commits while a verification job waits; two workers claim the same intent; a delivery call times out after the remote system accepts it; the email address hard-bounces between attempts; callbacks arrive in reverse order; the policy version changes during a deployment; and token redemption races across two application instances. Assert both external behavior and the evidence rows. One message with two decision records is a defect even if the user receives only one copy.

In staging, adapters should be deterministic fakes that can return accepted, rejected, timeout-after-accept, and delayed-callback outcomes. Production deployment needs a policy-version compatibility rule: workers running old code must either understand the new version or stop claiming those decisions. Alert on growing undecided intents, stale worker leases, suppression ingestion lag, and attempts stuck in an unknown state. Aggregate delivery rates are useful, but they cannot reveal an eligibility check that never ran.

Keep the proof reproducible. A reviewer should be able to take one decision row, retrieve the immutable policy definition and referenced input versions, and obtain the same allow or deny result without contacting a delivery provider.

## The rejected shortcut and where it still fits

The rejected option is a single `users` row containing `email_opt_in`, `sms_opt_in`, and `last_message_status`, followed by a direct send from the signup route. It is attractive because reads are cheap and the implementation is short. It also loses history on every update, conflates user preference with address-level suppression, and cannot represent an ambiguous attempt without overwriting some earlier truth.

It still has a valid use case: an internal prototype using synthetic destinations, where no real person can receive a message and no compliance claim will be made. Once real contact data enters the flow, migration should happen before delivery is enabled. Backfill current state as an explicitly labeled import with an `observed_at` timestamp; do not invent consent timestamps or sources that were never captured.

For an e-commerce signup flow, the durable decision is straightforward: separate identity, eligibility, intent, and transport evidence; resolve mutable controls at dispatch time; and preserve versions rather than mutable summaries. **A sent receipt answers “what happened?” The four-record chain answers “why was it allowed?”**

## References

- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://www.rfc-editor.org/rfc/rfc8058
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- CTIA Messaging Principles and Best Practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
