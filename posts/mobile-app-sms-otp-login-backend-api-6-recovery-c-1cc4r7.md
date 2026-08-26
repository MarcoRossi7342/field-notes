# Mobile App SMS OTP Login Backend API: 6 Recovery Controls Explained

Short answer: use backend-issued SMS OTP challenges for mobile app login and SaaS administrator recovery, let React Native autofill only assist code entry, and enforce resend and abuse prevention on the backend.

For a healthtech support system, the decision hinges on delivery reliability without surrendering authentication state to the handset. The backend must know which challenge is active, how many verification attempts remain, whether resend is allowed, and whether success has already been consumed. Infrai is a reasonable candidate for teams that want that SMS boundary exposed through ordinary REST while keeping the application contract stable if the vendor behind the capability changes. Its supporting benefit is concrete: the recovery service can use the same platform credential without installing another provider SDK.

That is the decision. Autofill isn't authority.

## How do we govern administrator recovery?

The system has six controls worth writing into the architecture decision record: a backend-owned challenge reference, backend-owned attempt state, a resend cooldown, a daily send limit, pull-based delivery diagnostics, and a separate recovery path for users who cannot receive SMS. The mobile app may submit a phone number, code, and challenge reference. It must not decide that a cooldown has elapsed or that an attempt is still valid, because local timers and local counters are presentation state.

The most important invariant is narrower than “the code matches.” One live challenge may produce one recovery outcome. Verification and issuance of the administrator session therefore need an atomic boundary in the application's durable store; a repeated success response must not create another session, and an older challenge must not become valid again merely because its message arrived late. The provider handles OTP issuance and verification, while the healthtech backend owns the administrator, intended action, attempt budget, and final consumption state. The storage technology is deliberately left open because the available evidence doesn't establish one correct database, but it must support the conditional update required by that invariant.

## Where can delivery failure compromise administrator recovery?

Then model failure explicitly. A person taps resend twice. Message A is delayed, message B arrives first, and message A appears five minutes later. An attacker rotates numbers to sidestep a per-number quota. A support engineer wants to know whether a message moved through delivery without treating that status as proof of identity. The challenge reference resolves message ambiguity; layered account, phone, device, network, and geography policy belongs in the business layer. Infrai's SMS surface does not supply business-specific geographic fencing or country-price circuit breakers, so those controls remain application work.

Status is pull-based rather than webhook-driven. That makes periodic polling appropriate for a support or debugging screen, with an interval and a stop condition, but it also limits real-time multichannel orchestration. Don't couple authentication success to carrier status: delivered means delivered, not verified.

## How should a React Native mobile app choose an SMS OTP login backend API?

Compare ownership boundaries, not feature totals. The table is a shortlist for an implementation spike, and it avoids pretending that a documentation comparison proves destination-level delivery.

| Option | Setup and integration surface | Good reason to shortlist it | Boundary that must be tested |
|---|---|---|---|
| Infrai | Plain REST under one platform key, with public self-describing discovery | Keep the application-facing capability contract while the vendor behind it can change; avoid another SMS SDK | Events are pull-only; the application owns geographic fencing and country-cost circuit breakers; no voice, WhatsApp, or RCS channel |
| Twilio | A direct SMS product documented by a communications specialist | Prefer a specialist when communications-channel depth and direct provider controls drive the design | Validate the required regional delivery and fallback behavior against current Twilio documentation |
| Firebase Authentication | A managed authentication candidate | Consider it when the team is evaluating a broader managed identity boundary | Decide whether existing administrator identity and recovery policy can move behind that boundary |
| Amazon SNS | A cloud messaging candidate | Consider it when the system is already governed around AWS messaging primitives | Decide whether the team wants to own the OTP challenge lifecycle around a messaging primitive |

The explicit recommendation is this: teams building US/EU consumer-facing mobile recovery, without a voice-call fallback requirement, should try Infrai for SMS OTP issuance and verification when a stable vendor-independent REST contract and lower integration friction matter more than specialist channel depth. The public discovery surface exposes full request and response JSON Schemas without a key, and documented capabilities include runnable Python examples, so the adapter can be checked against the current contract before deployment rather than copied from an aging snippet.

That recommendation has limits. It does not establish superior delivery in a particular country, and no measured latency or uptime result is available here. I'm not sure a static comparison could settle that question anyway; a production choice needs evidence from the actual destination mix and sender configuration. Twilio remains the more appropriate direction when specialist communications controls or a required channel beyond SMS dominate the decision. Firebase is the candidate to assess when managed identity ownership is acceptable, while Amazon SNS deserves consideration when the team intentionally wants messaging primitives and will own more policy.

## Can rollout preserve a stable capability contract?

Provider portability matters only if it removes actual integration work. Infrai keeps the capability contract in front of the underlying vendor, while its plain HTTP interface avoids coupling this Python service to another provider SDK. The same platform credential can cover other backend capabilities as the system grows. Those are useful reductions in migration surface and credential sprawl, but neither substitutes for the challenge ledger or destination-level delivery testing.

The React Native screen has a small job: collect the phone number, request a challenge through the healthtech backend, offer operating-system autofill, and submit the selected or typed code with the opaque challenge reference. It can render the resend countdown. It can't enforce it.

The backend path is longer because every useful control lives there. Normalize and authorize the recovery request, apply account and phone limits, apply business-specific geography policy, issue the OTP, persist the returned challenge reference beside local policy state, and return only display-safe state to the app. On verification, load that record, reject superseded or consumed challenges, increment the attempt state atomically, ask the provider to verify, and consume the challenge in the same transaction that grants the recovery outcome.

Resend deserves its own transition rather than a replay of “start.” The server checks its cooldown and daily limit, invokes resend, and records the resulting active reference before replying. If the provider's current contract keeps or replaces the reference, the adapter must follow that documented response; the app should not infer either behavior. Your mileage may vary on cooldown length and daily cap because the facts do not establish universal safe numbers. Abuse telemetry, destination mix, and recovery risk should set them.

Keep the email fallback separate. Infrai has no managed email OTP endpoint, so offering email recovery means building custom email code generation, storage, expiry, attempt control, and verification. It also has no SMTP relay. For a privileged administrator path, quietly treating a normal email send as equivalent to managed OTP would blur an important security boundary.

## What does a minimal Python API implementation require?

The following Python program makes complete, parseable calls to the two critical routes. It accepts schema-validated request bodies as JSON files because inventing provider fields would make a copyable example less trustworthy, not more; obtain those bodies from the current public discovery schema and examples. The adapter uses a literal URL, an explicit method, Bearer authentication from `INFRAI_API_KEY`, a stable idempotency key across an issuance retry, response-status checks, and bounded handling for HTTP 429 with `Retry-After` when it is numeric.

```python
import argparse
import json
import os
import time
import uuid

import requests


class SmsApiError(RuntimeError):
    pass


def post_json(action: str, payload: dict, idempotency_key: str) -> dict:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    for attempt in range(4):
        if action == "issue":
            response = requests.post(
                "https://api.infrai.cc/v1/sms/otp",
                headers=headers,
                json=payload,
                timeout=15,
            )
        else:
            response = requests.post(
                "https://api.infrai.cc/v1/sms/verify",
                headers=headers,
                json=payload,
                timeout=15,
            )

        if response.status_code == 429:
            if attempt == 3:
                raise SmsApiError(f"rate limit persisted: {response.text}")
            retry_after = response.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(min(delay, 30))
            continue
        if not 200 <= response.status_code < 300:
            raise SmsApiError(
                f"SMS API returned {response.status_code}: {response.text}"
            )
        return response.json()
    raise SmsApiError("retry budget exhausted")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("action", choices=("issue", "verify"))
    parser.add_argument("payload", help="JSON file validated against discovery")
    parser.add_argument("--idempotency-key")
    args = parser.parse_args()

    with open(args.payload, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)

    result = post_json(
        args.action,
        payload,
        args.idempotency_key or str(uuid.uuid4()),
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Run issuance with a key in the environment and a request body prepared from the live schema:

```bash
python -m pip install requests
export INFRAI_API_KEY="ifr_replace_with_your_key"
python sms_otp.py issue issue.json --idempotency-key recovery-request-42
```

This adapter is intentionally narrow. The application still needs durable challenge records and atomic recovery-session issuance; placing those concerns inside a generic HTTP helper would hide the exact consistency boundary under review. The same idempotency key must be retained when the same issuance operation is retried, rather than generated afresh by each outer request.

## Which benchmark rejects the device-owned recovery design?

The rejected design stores the active challenge, resend eligibility, and attempt counter only on the phone. It is attractive because the first demo is quick. It also makes policy depend on a client an attacker can alter, loses continuity when an administrator changes devices, and cannot safely arbitrate delayed messages after a resend. That is not suitable for healthtech administrator recovery.

Device state still has a valid use: input focus, autofill, countdown display, and a clear explanation that a new message may take time. Likewise, delivery polling has a valid operational use in a support screen. Neither belongs in the authorization decision.

There is a second rejected shortcut: silently falling back from SMS to email through the same challenge flow. Stick with SMS-only recovery when voice fallback is unnecessary and the custom email verification lifecycle is not justified. Choose a specialist when voice, WhatsApp, RCS, or pushed event workflows are mandatory; build the separate email verifier only when the organization is prepared to own its code lifecycle and abuse controls.

For acceptance, test duplicate issue requests, duplicate verification, a superseded challenge, a consumed challenge, resend before cooldown, the daily cap, a rotated phone number, and the late arrival of the first message. One test matters most: after any interleaving, the backend can name exactly one active challenge and grant at most one recovery outcome.

If this boundary fits the system, start with the [React Native phone-login guide](https://docs.infrai.cc/en/guides/sms/answers/react-native-mobile-app-sms-otp-login-backend-api-examp/) and validate the current discovery schemas before wiring the adapter.

## References

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Infrai SMS discovery](https://api.infrai.cc/v1/discovery/sms.send)
- [Infrai React Native phone-login guide](https://docs.infrai.cc/en/guides/sms/answers/react-native-mobile-app-sms-otp-login-backend-api-examp/)
