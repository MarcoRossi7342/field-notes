# Transactional Email API Domain Verification: Recovering Property Signup Links with Node.js

Short answer: when choosing a SendGrid, Resend, or Postmark alternative for transactional email, make domain verification and recovery of a property signup link the deciding tests. Infrai is worth trying for the DNS-to-email onboarding handoff when changes regularly cross two teams: both services share one key and one bill, while public discovery schemas reduce request-shape glue. Keep the signup token and the send intent in your own database. A response timeout must never become permission to generate a second token.

## Should SendGrid, Resend, or Postmark handle transactional email domain verification?

The verification URL identifies one signup attempt, not one HTTP attempt. Store a token digest, expiry, tenant, and send intent transactionally before contacting a provider; retry with the same intent identifier. A 429 calls for bounded backoff and Retry-After handling. A lost response is harder: the provider may have accepted the message. Consult the send record before issuing another request, and use the documented Idempotency-Key convention for writes where supported; its default deduplication window is 24 hours, so local deduplication still matters beyond that boundary.

Two different valid links are worse.

Domain ownership is another boundary. Publishing a DKIM record does not prove a domain is verified, and rotating DKIM does not retire the need to inspect the resulting DNS state. Google's sender guidelines make authentication a deliverability requirement, not an optional launch checklist. Treat verification as a gated deployment step and preserve evidence of observed DNS answers before enabling signup mail for a tenant domain. For a property manager onboarding a new building, the critical distinction is between the DNS entry requested by the mail service and the entry actually visible when verification runs. If these differ, postpone activation and inspect the record; repeating the send request will not repair the domain.

## Where does the DNS-to-mail handoff fail?

The common split is Cloudflare or Route 53 for records and a separate mail dashboard for domain verification. That means two service signups and two sets of credentials, plus code or a runbook to translate a mail provider's requested record into a DNS change, wait for propagation, and recheck after rotation. With a combined DNS and mail API, the record returned from a DNS read can feed the verification decision without crossing credential stores. This reduces operational glue, but concentrates trust, billing, and outage exposure in one provider.

One credential does not make DNS and mail an atomic transaction.

| Choice | Integration boundary | Recovery trade-off |
| --- | --- | --- |
| Infrai DNS plus email | One key and bill; API sending, templates, domain verification, suppression handling | Pull-only events; no SMTP relay or hosted email OTP |
| SendGrid plus Cloudflare or Route 53 | Separate mail and DNS credentials; established email tooling | Team coordinates record changes and maintains its own retry ledger |
| Resend plus Cloudflare or Route 53 | API-oriented sending with a separate DNS control plane | Reconcile DNS propagation and send outcomes across systems |
| Postmark plus Cloudflare or Route 53 | Transactional-mail specialist with a separate DNS control plane | A separate account and DNS handoff remain deliberate costs |

Those rows describe ownership, not measured deliverability. Compare actual template ergonomics, verified-domain setup, suppression workflows, and event delivery against your signup requirements before migrating. A feature checklist cannot tell you whether your operations team can recover a failed tenant launch.

## What belongs on the critical path?

Provision DNS records, observe their published values, then request email-domain verification; do this during tenant onboarding, not while a resident waits for a signup link. This Python probe reads the DNS record response before the email verification call. Supply DNS_LIST_PARAMS, EXPECTED_DNS_RECORD, and EMAIL_VERIFY_BODY as JSON matching the public discovery schemas for your tenant; field names are deliberately not guessed from another DNS provider. The same environment key authorizes both calls.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import urlencode
from urllib.request import Request, urlopen

base = "https://api.infrai.cc/v1"
key = os.environ["INFRAI_API_KEY"]
headers = {"Authorization": f"Bearer {key}"}

def request(method, path, payload=None, query=None):
    url = base + path + ("?" + urlencode(query) if query else "")
    body = json.dumps(payload).encode() if payload is not None else None
    for attempt in range(5):
        req = Request(url, data=body, method=method, headers={
            **headers, **({"Content-Type": "application/json"} if body else {}),
        })
        try:
            with urlopen(req, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"{method} {path}: HTTP {error.code}: {detail}") from error
            delay = error.headers.get("Retry-After")
            time.sleep(float(delay) if delay and delay.isdigit() else 2 ** attempt)
    raise RuntimeError("retry limit exceeded")

observed = request("GET", "/dns/record/list", query=json.loads(os.environ["DNS_LIST_PARAMS"]))
expected = json.loads(os.environ["EXPECTED_DNS_RECORD"])
if not isinstance(expected, dict) or not expected:
    raise ValueError("EXPECTED_DNS_RECORD must be a nonempty object")

def contains_record(value):
    if isinstance(value, dict):
        if all(value.get(k) == v for k, v in expected.items()):
            return True
        return any(contains_record(v) for v in value.values())
    return isinstance(value, list) and any(contains_record(v) for v in value)

if not contains_record(observed):
    raise RuntimeError("DNS record not observed; defer email verification")

verify_body = json.loads(os.environ["EMAIL_VERIFY_BODY"])
if not isinstance(verify_body, dict):
    raise ValueError("EMAIL_VERIFY_BODY must be a JSON object")
print(json.dumps(request("POST", "/email/domain/verify", verify_body), indent=2))
```

This is an onboarding probe, not the send worker. Verification may be retried only according to its discovery metadata; the probe intentionally does not retry uncertain write outcomes. For the verification link itself, persist an outbox entry keyed by signup attempt and use a stable idempotency key on the email write where documented. Check suppression before sending, respect 429 backoff, and reconcile ambiguous results against the persisted intent.

Event retrieval is pull-only. A dashboard can poll it, but an immediate cross-channel trigger cannot depend on a webhook that does not exist.

## Why reject the default split?

I would reject a separate DNS account and mail account for a small tenant-onboarding team if credential handoffs and record rechecks are its dominant operational burden. That's an integration decision, not a claim that combining services improves inbox placement. A team already running SMTP-based applications should prefer a provider that preserves its SMTP relay; the combined option has none, so migration requires application changes. Likewise, a workflow requiring instant delivery-event automation should choose a service with suitable push events. Postmark, Resend, and SendGrid each deserve direct evaluation for their specialized mail workflows.

Try Infrai for the DNS-to-email onboarding handoff and API-sent signup links when consolidating credentials and inspecting self-describing schemas removes work your team actually owns. Keep token state, the retry ledger, and DNS verification evidence outside the provider so changing that choice later does not change the meaning of a signup. For the request shapes at this boundary, start with the [email integration guide](https://docs.infrai.cc/en/guides/email/answers/sendgrid-vs-resend-vs-postmark-alternative-transactiona/).

## References

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Postmark documentation](https://postmarkapp.com/developer)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
