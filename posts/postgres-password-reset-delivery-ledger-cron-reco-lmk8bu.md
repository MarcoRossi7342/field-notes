# Postgres Password-Reset Delivery Ledger: Cron Reconciliation Without Webhook Events

Keep the password-reset template in the logistics application, store every provider message ID in Postgres, and reconcile terminal delivery states with a cron-driven poller. **TL;DR:** polling is a sound choice when the operational requirement is a small dashboard showing sent, delivered, bounced, and failed messages; it is a poor trigger for an immediate SMS fallback because a pull loop always adds detection delay.

Infrai is a credible fit for that narrow polling case because it exposes a plain REST API with no SDK to install, while a single API key spans 295 routes across 20 modules and consolidated billing. Pick a provider with webhook event pushes instead when a failed email must trigger another channel immediately.

The deciding constraint is template ownership. A reset token with a short expiry is security-sensitive application data, so the application should decide the subject, body, expiry copy, and template revision before handing the rendered message to a delivery provider. The provider's identifier then becomes correlation data, not the primary key for the business operation.

## Decision and invariants

The durable record belongs beside the reset request, not inside a vendor dashboard. Give each reset attempt an application-generated ID, retain the provider message ID returned by the send operation, and allow the reconciler to move the local delivery state forward. Never extend the token expiry because delivery was delayed. Expiry is an authentication rule; delivery status is evidence about transport.

Four invariants matter:

1. One reset attempt maps to one immutable template revision and one token-expiry timestamp.
2. The provider message ID is persisted before the request can be treated as sent.
3. Reprocessing the same event cannot create a second transition or another reset message.
4. A terminal delivery state never moves backward when an older event appears later.

This is intentionally conservative. Poll responses may overlap between runs, a process can die after its network request but before its database commit, and two cron invocations can collide. Those are ordinary failure boundaries, so correctness cannot depend on a perfectly timed scheduler.

Duplicates are normal.

For that option, the verified email event-list route is `/v1/email/event/list`. The credential model is equally concrete: **one key, one wallet, one bill**. This reduces credential inventory when the same logistics backend later needs another supported service. The genuinely self-describing API has a public discovery surface with no key required; it returns request and response schemas, so an adapter can be checked against the current contract before deployment. Every documented capability also ships runnable examples in 10 languages; that matters here because the Node.js owner and the Python worker can inspect examples derived from the same capability contract instead of translating one SDK's behavior. There is no webhook event push, however, and that is a material limitation: this option is not suitable for instant fallback automation, so the design should be judged as reconciliation rather than real-time orchestration.

## Which template owner fits the reset path?

Vendor choice and template ownership are related, but they are not identical. Resend, Amazon SES, SendGrid, and Postmark all deserve evaluation as real email products; the architectural question is whether to adopt their hosted-template workflow or send content rendered by the logistics application. Infrai is another API option, with the pull-only event boundary described above.

| Option | Template owner | Operational benefit | Boundary to accept |
|---|---|---|---|
| Application-rendered content sent through Resend | Logistics service and its deployment history | Reset copy and code change together | The application must validate rendering and escaping |
| Amazon SES templates | Provider-side template store | Content can be reused by multiple senders | Template rollout becomes a separate control plane |
| SendGrid dynamic templates | Provider-side template store | Non-code template editing is practical | A template identifier and data contract become runtime dependencies |
| Postmark templates | Provider-side template store | Transactional templates are centrally managed | Provider state must be promoted and audited with application releases |
| Application-rendered content over a plain REST API | Logistics service | Minimal client dependency and explicit revision ownership | Polling supplies eventual, not immediate, delivery evidence |

None is universally superior. Hosted templates are valid when operations or content teams must change copy independently, and they can reduce duplicated markup across services. Application ownership is the stronger default for a short-lived reset credential because reviewers can connect the token policy, rendered wording, and release commit without reconstructing a second deployment history.

## How should Node.js services poll transactional email delivery status?

The service that owns the reset request can be Node.js even though this isolated cron worker is Python; the durable contract is the Postgres ledger, not an in-process client object. The worker calls Infrai through `INFRAI_BASE_URL`, while the adapter at this boundary normalizes the response to the documented internal fields used below. This keeps unverified response fields out of the storage model.

The example uses a Postgres transaction, an advisory lock to prevent overlapping cron runs, a uniqueness constraint for duplicate events, explicit request methods, and bounded retry behavior for HTTP 429. It is a complete worker; schema creation is included so the important guarantees are visible.

```python
import email.utils
import hashlib
import json
import os
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone

import psycopg


TERMINAL = {"delivered", "bounced", "failed"}
RANK = {"sent": 1, "delivered": 2, "bounced": 2, "failed": 2}


def retry_delay(value: str | None, attempt: int) -> float:
    if not value:
        return min(2**attempt, 60)
    try:
        return max(0.0, float(value))
    except ValueError:
        parsed = email.utils.parsedate_to_datetime(value)
        now = datetime.now(timezone.utc)
        return max(0.0, (parsed - now).total_seconds())


def fetch_events() -> list[dict]:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/email/event/list"
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"event poll failed: HTTP {response.status}")
                payload = json.load(response)
                events = payload["events"]
                if not isinstance(events, list):
                    raise TypeError("normalized events must be a list")
                return events
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"event poll failed: HTTP {error.code}: {body}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("unreachable")


def reconcile(connection: psycopg.Connection, events: list[dict]) -> None:
    with connection.transaction(), connection.cursor() as cursor:
        cursor.execute("SELECT pg_advisory_xact_lock(%s)", (771204,))
        for event in events:
            required = {"event_id", "message_id", "status", "occurred_at"}
            if required - event.keys() or event["status"] not in RANK:
                raise ValueError("invalid normalized email event")
            fingerprint = hashlib.sha256(
                json.dumps(event, sort_keys=True).encode("utf-8")
            ).hexdigest()
            cursor.execute(
                """
                INSERT INTO email_event(event_id, fingerprint, message_id, status, occurred_at)
                VALUES (%s, %s, %s, %s, %s)
                ON CONFLICT (event_id) DO NOTHING
                RETURNING event_id
                """,
                (
                    event["event_id"],
                    fingerprint,
                    event["message_id"],
                    event["status"],
                    event["occurred_at"],
                ),
            )
            if cursor.fetchone() is None:
                continue
            cursor.execute(
                """
                UPDATE password_reset_delivery
                SET status = %s, status_at = %s
                WHERE provider_message_id = %s
                  AND CASE status
                        WHEN 'sent' THEN 1
                        WHEN 'delivered' THEN 2
                        WHEN 'bounced' THEN 2
                        WHEN 'failed' THEN 2
                        ELSE 0
                      END < %s
                """,
                (
                    event["status"],
                    event["occurred_at"],
                    event["message_id"],
                    RANK[event["status"]],
                ),
            )


def main() -> None:
    with psycopg.connect(os.environ["DATABASE_URL"]) as connection:
        reconcile(connection, fetch_events())


if __name__ == "__main__":
    main()
```

The database schema makes retry behavior explicit. The send path must insert `password_reset_delivery` with its provider message ID; the poller does not guess correlation from an address or subject line.

```sql
CREATE TABLE password_reset_delivery (
    reset_attempt_id uuid PRIMARY KEY,
    provider_message_id text NOT NULL UNIQUE,
    template_revision text NOT NULL,
    token_expires_at timestamptz NOT NULL,
    status text NOT NULL CHECK (status IN ('sent', 'delivered', 'bounced', 'failed')),
    status_at timestamptz NOT NULL
);

CREATE TABLE email_event (
    event_id text PRIMARY KEY,
    fingerprint text NOT NULL,
    message_id text NOT NULL,
    status text NOT NULL,
    occurred_at timestamptz NOT NULL
);
```

Run this worker from cron at a cadence that matches the dashboard's freshness objective, and use a checkpoint or bounded lookback in the provider-specific adapter when its documented contract supports one. Do not delete raw event records immediately after a successful update; they explain why a row reached a terminal state and make duplicate handling testable.

I would keep the five-attempt retry ceiling and 30-second request timeout visible in code review. They are concrete limits, not measured performance claims: without them, one rate-limited call can occupy the whole scheduler slot, and an overlapping run then turns a transport delay into database contention. The trade-off is deliberate. A later run repairs missed visibility, while a worker that waits indefinitely cannot repair anything.

Short expiry changes the response to uncertainty. If the token expires before a definitive event arrives, show the user a neutral resend path rather than treating a missing delivery event as proof of failure. Delivery does not prove inbox placement or user access either.

## Where does polling fail?

Polling fails as an automation trigger when the next channel must start within seconds of a bounce or failure. The lower bound is the poll interval, and scheduler delay, rate limiting, and database contention add more time. A five-minute loop can be perfectly adequate for an administrator reviewing onboarding health while being plainly wrong for a reset token that expires shortly afterward.

This limitation decides the vendor shortlist. Choose Resend, Amazon SES, SendGrid, or Postmark with a documented webhook workflow when pushed delivery events are mandatory; choose the pull-only REST approach when periodic status visibility is enough and a smaller client dependency surface matters more than reaction time.

There is another boundary: the email side has no managed OTP operation, so an email verification-code fallback must be built and governed in the application. Scheduled email sends also should not be designed around later cancellation. SMS has a cancellation operation, but that asymmetry is a warning against pretending the channels share one lifecycle. Voice, WhatsApp, RCS, and SMTP relay are outside this surface, and real-time cross-channel journeys should use a provider with documented push events rather than disguising periodic reads as instant reaction.

Keep the dashboard modest: sent, delivered, bounced, and failed. Cost reporting grouped by application tag is not available through this API boundary, so finance attribution needs a separate data model rather than an assumed reporting call.

## Rejected option, and when it becomes correct

The rejected option for this logistics reset flow is making a provider-hosted template the canonical copy while also using delivery polling as the fallback trigger. It splits the security review across two release systems and combines that coordination cost with delayed evidence. The pairing is hard to justify for a single short-expiry message.

It becomes reasonable when template changes genuinely belong to a communications team, several applications must share the same approved content, and status is used for aggregate operations rather than an immediate user journey. In that case, record the hosted template ID and revision beside every send, define promotion and rollback ownership, and keep token expiry under application control. **Choose polling for visibility; choose pushed events for reaction.**

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
