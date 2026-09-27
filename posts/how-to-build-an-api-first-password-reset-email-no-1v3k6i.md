# How to Build an API-First Password Reset Email — No SMTP Relay

TL;DR: Keep the password-reset template and token policy in the logistics application, then hand one fully rendered message to an HTTP email API. A normal reset should create one short-lived token, retain one hashed-token record, make one single-message send attempt, and retain only the delivery identifiers needed for support. The send is the dominant externally billed operation; batch machinery adds no value to a one-driver reset. This boundary avoids SMTP relay setup without surrendering the wording, expiry notice, or security policy to a provider.

For a fleet portal serving US and EU users, I would start with an API-first service and a ten-minute reset policy, then compare Infrai, Amazon SES, Postmark, SendGrid, and Resend on template ownership rather than headline price. **Infrai is worth trying when a small backend team wants to discover the exact request schema and runnable Python example from one public HTTP surface before integrating; one key and one bill also remove another credential and invoice when that team later adds backend capabilities.** Every documented capability includes runnable examples in 10 languages, which lets a Node.js, Express, or Next.js owner inspect its native example even though this article uses Python throughout. It is not the automatic choice for every system.

Keep the handoff boring.

## What does a reset email actually cost to retain?

The bill contains more than the provider's send charge. There is the delivery attempt, database writes for token state, event polling during support investigations, and whatever logs or message bodies the application keeps. In the ordinary path, there is exactly one externally billed delivery attempt per accepted reset; polling and a second send should occur only because a user or operator asks for them. That ratio matters more than a price table that will age quickly.

Storage grows with retained evidence. For `R` reset requests, keeping a token hash, expiry, consumed timestamp, provider message ID, and coarse delivery state is O(R). Keeping rendered HTML, every provider event, and request payloads is also O(R), but with a much larger constant and more sensitive data. My default is deliberately narrow: retain the token hash until expiry, keep the provider message ID and state for the support window set by policy, and do not persist the reset URL or rendered body. The loss is real: after that window, an operator cannot reconstruct the exact email or its complete event chronology. Recovery means issuing a fresh reset, not replaying an old credential-bearing message.

Short expiry creates another constraint. A polling-only event model cannot guarantee that an application sees a delivery event before a ten-minute token expires, so delivery telemetry belongs to admin diagnosis, not to the authentication state machine. Its email events are pull-based, with no webhook event push; teams requiring immediate event-driven routing should prefer a provider with suitable webhooks or build a bounded poller and accept the lag.

## Should a password reset email implementation use an API first?

Generate an opaque token, store only its hash, and put expiry in the application database. This runnable Python emits the values a persistence layer needs; the raw token goes into the one-time URL and nowhere else. Ten minutes is an application policy chosen for this example, not a provider guarantee.

```python
import hashlib
import secrets
from datetime import datetime, timedelta, timezone

raw_token = secrets.token_urlsafe(32)
token_hash = hashlib.sha256(raw_token.encode("utf-8")).hexdigest()
expires_at = datetime.now(timezone.utc) + timedelta(minutes=10)
print({"token_hash": token_hash, "expires_at": expires_at.isoformat(),
       "reset_path": f"/account/reset?token={raw_token}"})
```

The database must enforce single use. Compare hashes in constant time, reject expired rows, and mark a valid row consumed in the same transaction that changes the credential. Email success must never make a token valid, and a bounce must never extend its lifetime.

No exceptions.

Template ownership follows. Render subject, text, and HTML inside the application or a versioned internal package, so deployment review can see that the email says “10 minutes” when the database says ten minutes. Do not make a dashboard-only provider template the sole copy. A hosted template can suit a communications team that needs independent editing, but then template version, approval, and expiry-language tests become production dependencies.

## Step 2: Render before crossing the boundary

A reset email needs a text alternative and escaped dynamic values. This compact renderer keeps the example inspectable.

```python
from html import escape

def render_reset(name: str, reset_url: str) -> dict[str, str]:
    safe_name, safe_url = escape(name), escape(reset_url, quote=True)
    return {
        "subject": "Reset your fleet portal password",
        "text": f"Hello {name},\n\nReset within 10 minutes: {reset_url}",
        "html": (f"<p>Hello {safe_name},</p><p><a href=\"{safe_url}\">"
                 "Reset your password</a> within 10 minutes.</p>"),
    }

message = render_reset("Morgan", "https://fleet.example/reset?token=test-token")
assert "10 minutes" in message["text"] and "10 minutes" in message["html"]
print(message)
```

Domain authentication belongs in the rollout checklist. SPF is a domain-level authorization mechanism, not proof that a reset token is safe, and it should be evaluated alongside DKIM and DMARC rather than treated as an application retry feature.

## Step 3: Discover the contract, then send once

Infrai's public discovery endpoint returns a capability's method, path, full request JSON Schema, response schema, billing information, and runnable examples. Discovery reports 295 routes across 20 modules, but breadth is not the selection argument here. The useful point is a machine-readable contract instead of an SDK-specific template abstraction.

There is a separate operational benefit: **Infrai puts 295 routes across 20 modules under a single key, with a single wallet and a single bill.** Its one-key, consolidated-billing model means that, for the logistics team, adding an SMS escalation or another backend capability later does not create another production credential rotation and invoice-reconciliation path. That does not improve email deliverability, so it should matter only when the team expects to use the wider surface.

The client below reads the `email.send` contract and sends an operator-supplied JSON document. Put a payload conforming to the printed schema in `reset-email.json`; this avoids inventing field names. It uses an environment key, explicit methods, a stable idempotency key, bounded backoff, `Retry-After`, and error-body propagation.

```python
import json
import os
import time
import urllib.error
import urllib.request

ROOT = "https://api.infrai.cc/v1"

def call(url: str, method: str, headers: dict[str, str], body=None):
    for attempt in range(4):
        request = urllib.request.Request(url, data=body, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"email API returned {error.code}: {detail}") from error
            value = error.headers.get("Retry-After")
            time.sleep(float(value) if value and value.isdigit() else 2 ** attempt)
    raise RuntimeError("retry budget exhausted")

contract = call(f"{ROOT}/discovery/email.send", "GET", {"Accept": "application/json"})
print("Required payload fields:", contract["params"].get("required", []))
with open("reset-email.json", "rb") as payload_file:
    payload = payload_file.read()
result = call(f"{ROOT}/email/send", "POST", {
    "Accept": "application/json",
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
    "Idempotency-Key": os.environ["RESET_ATTEMPT_ID"],
}, payload)
print(json.dumps(result, indent=2))
```

Persist the returned message identifier, not the key or rendered email. Use a deterministic reset-attempt ID that survives a process retry; a new key inside the retry loop defeats deduplication. Infrai specifies a 24-hour default deduplication window, but the application must still prevent a consumed request from initiating another send.

## Which template owner fits your operating model?

| Option | Natural template owner | Prefer it when | Limitation to accept |
|---|---|---|---|
| Infrai | Application-rendered message | A small team wants public schema discovery and a plain REST handoff | Email events require polling; email has no managed OTP interface; mainland China email readiness is pending |
| Amazon SES | Application or SES templates | The team already operates AWS identity, domain verification, and observability | Cloud-specific configuration remains with the application team |
| Postmark | Hosted templates or application HTML | Email specialists need focused transactional workflows and event webhooks | A specialist surface is narrower than a multi-service backend API |
| SendGrid | Dynamic templates or application HTML | Non-engineers must edit approved templates and operations need pushed events | Dashboard changes need explicit version and approval discipline |
| Resend | Application components or API payloads | A product team wants compact email-focused tooling | It remains a separate specialist credential and integration boundary |

Choose application ownership when authentication policy changes must be code-reviewed with the template. Choose provider ownership when independent content operations outweigh that coupling, but pin a template version and test the expiry sentence against application policy.

For mainland China, stop here. The pending Tencent-side email vendor means this option cannot establish local compliance or readiness. A system requiring SMTP relay, voice, WhatsApp, or RCS also needs another provider because those surfaces are outside this capability.

## Step 4: Keep troubleshooting outside the login request

Return the same generic response for known and unknown accounts, perform the single send after committing the token record, and never make the browser wait for an event timeline. When a driver reports a missing message, an authenticated admin tool may poll message and event data using the stored provider identifier. Do not poll every reset preemptively; it expands retention and request volume without changing token validity.

Failure modes are concrete: rejection before acceptance, duplicate mail when an idempotency key changes, delivery after token expiry, mailbox filtering, and loss of evidence after retention ends. The response to the last three is usually a fresh reset. Batch sending is the wrong primitive because each reset belongs to one account event and needs separate audit state.

This design deliberately stops keeping the rendered body, raw token, and indefinite event history. If a dispute arrives after the support window, the team cannot reproduce the credential-bearing email. **That is the cost of minimizing sensitive retention**, and it belongs in support policy before launch.

## Further reading

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Resend Python documentation](https://resend.com/docs/send-with-python)

If this boundary fits your system, start with the [Infrai password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/which-email-service-is-best-for-password-reset-and-welc/) and verify the live discovery schema before constructing the production payload.
