# 5 Transactional Email API Tests — Small SaaS GDPR Report Delivery

**TL;DR:** Keep the report template, rendered message, and attachment manifest in the application-controlled data layer; give the mail service only a time-bounded delivery job. For a small EU media SaaS, that choice matters more than a marginal price difference because it determines whether a regenerated audience report is reproducible, whether a retry can silently send different bytes, and whether erasure can be demonstrated. Evaluate Postmark, Resend, Mailgun, or a simpler API against that boundary rather than allowing any provider's template model to define it.

A generated media report is not an ordinary welcome message. The attachment may contain campaign names, audience segments, contact details, or other personal data, while the email body identifies the recipient and explains the reporting period. The architecture therefore has two storage problems: preserving enough evidence to explain what was sent, and avoiding indefinite duplication of the payload. Cheap submission is a weak decision criterion if those two responsibilities remain vague.

That is the trap.

## 1. Why should the application own the template?

A vendor-hosted template is convenient until a report must be reproduced after copy, localization, or merge logic has changed. An application-owned template can be versioned with the rendering code, reviewed beside schema changes, and identified in a durable manifest. This is a custody decision, not a stylistic preference.

The manifest should identify the template version, report object version, recipient, and a digest of the exact attachment bytes. It should not become a second copy of the report. Object storage supplies the immutable payload; the relational record supplies searchable delivery state; the mail adapter receives a bounded command.

**The retry key belongs to the report-delivery intent, not to an HTTP attempt.** Without that rule, a timeout followed by an eager retry can create two accepted submissions. Conversely, treating every timeout as final can suppress a report that was never accepted. Persist `pending`, submit once per intent, reconcile provider events, and move ambiguous outcomes into an explicit review or delayed-retry state. Exactly-once delivery is not something SMTP promises.

| Concern | Application-owned artifact | Provider-facing artifact | Failure mode to test |
|---|---|---|---|
| Message copy | Versioned template and locale | Rendered subject and body | A deploy changes copy during retry |
| Report file | Immutable object version and SHA-256 digest | Attachment bytes | The object is overwritten under the same key |
| Recipient | Normalized address plus lawful-purpose metadata | Destination address | An old job survives an erasure request |
| Delivery state | Intent ID and event history | Provider message ID | Submission succeeds but the response is lost |

SPF does not solve any of these storage questions. RFC 7208 defines authorization for hosts using a domain in SMTP identities; it is one part of sender authentication, not proof that a particular attachment reached its intended human. Keep authentication status in operational telemetry, but do not confuse it with business-level delivery evidence.

## 2. Make the attachment immutable before submission

Render the report first. Then hash it, record its byte length and media type, and attach that exact object version. If rendering happens inside the send attempt, two retries can carry different reports while sharing one logical intent, especially when source analytics continue to change.

No overwrite.

The following Python boundary is deliberately small. It validates the artifact that the application has already produced and passes neutral message data to an adapter; it does not place templates or report generation inside a provider SDK.

```python
from dataclasses import dataclass
from hashlib import sha256
from pathlib import Path
from typing import Protocol


@dataclass(frozen=True)
class DeliveryIntent:
    intent_id: str
    recipient: str
    template_version: str
    report_path: Path
    expected_sha256: str


class MailAdapter(Protocol):
    def submit(
        self, *, intent_id: str, recipient: str, subject: str,
        body: str, attachment_name: str, attachment: bytes
    ) -> str: ...


def submit_report(intent: DeliveryIntent, mailer: MailAdapter) -> str:
    payload = intent.report_path.read_bytes()
    actual_digest = sha256(payload).hexdigest()
    if actual_digest != intent.expected_sha256:
        raise ValueError("report bytes changed after the intent was created")

    return mailer.submit(
        intent_id=intent.intent_id,
        recipient=intent.recipient,
        subject="Your media performance report",
        body="Your requested report is attached.",
        attachment_name="media-report.pdf",
        attachment=payload,
    )
```

The adapter must also enforce a configured attachment-size ceiling before allocating or encoding a large object. The ceiling cannot be inferred from a generic standard because the relevant accepted-message and encoded-payload limits depend on the selected service and downstream mail path. Record the configured limit, test a payload immediately below it, and reject an oversized report before submission. A link to an access-controlled download is a separate product decision, with its own expiry and authorization semantics; it is not an invisible substitute for an attachment.

## 3. How should a small SaaS test a transactional email API?

Postmark, Resend, and Mailgun all document APIs for sending email, but their attachment fields, template workflows, regional or account configuration, event formats, retention controls, and payload limits must be checked in their current primary documentation during procurement. Those boundaries can change. A generic "simple email API" deserves the same examination; fewer features do not remove controller obligations or delivery failure modes.

A fair proof of concept sends the same application-rendered fixture through each adapter and scores observable behavior. Do not move the fixture into three vendor template systems, because that changes the ownership question being tested. Do not use the production report either. Synthetic recipient and report data are enough to verify MIME construction, attachment naming, Unicode handling, event correlation, suppression behavior, and deletion procedures.

Test the boundary.

| Decision test | Evidence to collect | Reject when |
|---|---|---|
| Template custody | Exportable source and version ID in the application repository | A correct resend requires mutable dashboard state |
| EU data handling | Contract terms, subprocessors, transfer mechanism, and selected processing region | The team cannot document where relevant message data is processed |
| Retry reconciliation | Stable intent metadata plus accepted, delivered, bounced, and complained events | An ambiguous submission cannot be correlated without recipient search |
| Attachment limits | Current documented limit and boundary test result | Oversize behavior is undocumented or only discovered in production |
| Erasure and retention | Written retention schedule and tested deletion workflow | Logs or payloads persist without a stated purpose and deadline |
| Exit cost | Adapter fixture passes against a second implementation | Business templates or event meaning cannot be exported |

This comparison intentionally produces no universal winner. One service's dashboard templates may suit a marketing-owned workflow and be the wrong boundary for an engineering-owned generated report. Another may expose a pleasant API while failing a particular organization's region, retention, or contract requirements. Price belongs in the final scorecard as expected monthly cost at the team's actual volume, including retries and support, but it should not erase a failed governance or payload test.

## 4. Minimize data without destroying evidence

GDPR Article 5 includes purpose limitation, data minimization, storage limitation, integrity, and confidentiality. Article 28 sets requirements for processing by a processor, while Articles 44 onward govern transfers of personal data to third countries or international organizations. Those obligations are broader than selecting an EU endpoint. The organization still needs a documented purpose, roles, contract, transfer analysis where applicable, access controls, and retention schedule.

Separate evidence by sensitivity and useful lifetime. A compact delivery ledger can retain an intent ID, template version, attachment digest, timestamps, outcome category, and provider correlation ID without retaining the attachment itself. The address may need tighter access and earlier deletion or pseudonymization than aggregate operational metrics. The source report object should follow the product's report-retention policy, while transient staging files should expire quickly after the delivery reaches a terminal state. Exact periods are organizational decisions grounded in purpose and legal requirements, so inventing a universal number would be misleading.

Logging is an easy leak. Never put attachment bytes, rendered HTML, signed download URLs, or full API responses into ordinary application logs. Redact recipient data where it is not needed, restrict access to the delivery ledger, and make event ingestion reject or quarantine events that fail signature verification according to the selected provider's documented scheme. Metrics should describe counts and latency distributions; traces should carry the internal intent ID rather than message content.

## 5. Roll out with a reversible adapter

Start with shadow rendering: create and hash synthetic reports without sending them, then compare manifests across deploys. Next, send to controlled test inboxes and exercise accepted, bounced, complained, delayed, duplicate-event, out-of-order-event, and lost-response cases. Event handlers must be idempotent because webhook delivery can be retried, and state transitions should reject regressions such as `delivered` returning to `queued` merely because an older event arrived late.

After a limited production cohort, compare the internal intent ledger with provider events and investigate gaps. Track submission latency, ambiguous submissions, terminal outcomes, attachment rejections, and the age of pending jobs. Rollback should switch adapters while preserving the same intent IDs, template versions, and immutable report objects.

The final rule is compact: **own the bytes and their meaning; rent transport.** That boundary keeps a generated media report reproducible, makes retention review possible, and lets the team change delivery services without redefining what was sent.

## Sources

- https://datatracker.ietf.org/doc/html/rfc7208
- https://www.rfc-editor.org/rfc/rfc5321
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://postmarkapp.com/developer/api/email-api
- https://resend.com/docs/api-reference/emails/send-email
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages
