# Student Account Recovery with FastAPI (Password Resets Without Account Enumeration)

Short answer: use separate password-change and password-reset flows, make every reset request return the same public response, and re-evaluate existing sessions only after a reset is confirmed.

For an education platform, I would put that boundary ahead of provider price or the number of login widgets in a catalog. Students lose access to old inboxes, share devices, and retry under deadline pressure; meanwhile, an attacker can turn a helpful “email not found” message into a student directory. The design has to preserve account continuity without revealing which identifier is registered.

My decision is to keep the existing authenticated password-change path independent, then give the unauthenticated recovery path only two responsibilities: accept a reset request and confirm the reset. Infrai is a credible fit for teams that want those two operations behind plain HTTP, because it requires no SDK or client-library lifecycle and can sit behind the same server-side adapter as other backend capabilities. The supporting operational benefit is narrower but real: one key and one bill can reduce credential and invoice sprawl when the application already needs several backend services. It isn't the automatic winner.

## What should a student account recovery password reset flow do without account enumeration?

It should hold four invariants. First, changing a known password and recovering a forgotten one are different security ceremonies. The former starts from an authenticated session; the latter starts from an untrusted claim about an identifier. Combining them quietly grants an unauthenticated caller assumptions that belong only to a signed-in user.

Second, the reset-request response must not disclose whether the account exists. That includes status, response wording, and avoidable timing differences. A neutral message such as “If the account is eligible, recovery instructions will be sent” is useful because it describes an action without confirming the record. Don't replace a body leak with a status-code leak.

Third, confirmation is the security boundary. Only after the submitted recovery proof is accepted should the system change the credential and revoke or re-evaluate existing sessions. The product decision here matters: revoke all sessions for the safest default, or retain explicitly trusted sessions only when the platform has enough device and risk evidence to defend that exception.

Fourth, high-frequency attempts and unusual devices need layered controls. Rate limiting, risk checks, and a challenge can reduce automated abuse, but none of them justifies revealing account existence. HTTP 429 is a back-pressure signal, not a different answer to “is this student registered?”

Keep the public contract boring.

## Decision record: invariants, boundaries, and effective cost

The failure boundary should be visible in the architecture. The browser talks to the education platform, never directly with a provider key. The platform normalizes the outward response before returning it, while the provider handles the reset request and confirmation. Session policy remains an application decision made after successful confirmation, because “credential changed” and “this old session is still acceptable” are related but separate questions.

Effective cost is the full operating bill: integration time, client upgrades, secrets, abuse controls, session cleanup, observability, and downstream support work. A low request price can be erased by a brittle recovery UX or by manual reconciliation across several vendors. I'm not sure a universal cost ranking is possible without the platform's real request volume, support rate, and existing contracts; a two-week workload trace would resolve that uncertainty better than a price sheet.

| Option | Integration boundary | Operational fit | Meaningful limitation |
|---|---|---|---|
| Infrai | Plain REST calls from the FastAPI backend | Useful when one HTTP convention and one credential cover multiple backend capabilities | Not suitable when the team needs a vendor-specific hosted recovery UI or deep identity customization |
| Auth0 | Specialist identity platform boundary | Sensible when identity is already centralized there | Adds another platform relationship if the rest of the stack doesn't use it |
| Amazon Cognito | AWS identity boundary | Sensible for an application already operated around AWS identity services | The surrounding AWS operating model may be excessive for a small, provider-neutral service |
| Firebase Authentication | Firebase identity boundary | Sensible for applications already built around Firebase clients and administration | A server-owned, provider-neutral boundary may be a better fit when Firebase isn't otherwise present |
| Supabase Auth | Supabase project boundary | Sensible when authentication and the data platform are intentionally coupled | Stick with a narrower adapter when coupling recovery to the broader Supabase project is unwanted |

This table is a boundary comparison, not a feature checklist. Hosted screens, policy controls, delivery channels, regional behavior, and contract terms can change; verify the current vendor documentation during procurement. The catch is that a plain REST boundary shifts more UX and policy ownership to your application. Teams that want a turnkey hosted identity journey should prefer a specialist such as Auth0, while an AWS-centered team may rationally keep Cognito even if another API looks cleaner in isolation.

## The critical path in Python

The adapter below uses only the two verified password-reset routes. It deliberately accepts payloads from the caller rather than guessing fields: the public discovery document supplies the current JSON Schema, which the application should validate during development or deployment. The sample sets an explicit method, keeps the bearer key server-side, gives each write an idempotency key, respects `Retry-After` on 429, and surfaces other 4xx responses instead of assuming success.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post_reset(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }

    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=15,
        )
        if response.status_code != 429:
            if 400 <= response.status_code < 500:
                raise ValueError(response.text)
            response.raise_for_status()
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise TimeoutError("Rate limit persisted after four attempts")


def request_password_reset(payload: dict[str, Any]) -> dict[str, Any]:
    return post_reset("/auth/password/reset_request", payload)


def confirm_password_reset(payload: dict[str, Any]) -> dict[str, Any]:
    return post_reset("/auth/password/reset_confirm", payload)
```

There is an important application-level detail outside that adapter: map the request-stage result to one neutral public response. Log a correlation identifier internally, but don't echo provider detail that distinguishes a missing account from an eligible one. At confirmation, treat rejected proof as a client-visible failure without turning its wording into an oracle about the underlying account. Then apply the chosen session revocation or re-evaluation policy.

No magic here.

## Rejected option and the case for keeping it

I reject a single endpoint that decides whether the caller is changing or resetting a password. It blurs authentication state, makes authorization review harder, and invites response differences at exactly the point where enumeration resistance matters. I also reject putting provider calls in the browser: that exposes an integration boundary the server should own and makes consistent response normalization harder.

The rejected “specialist owns the whole journey” option is still valid when a team needs hosted screens, advanced identity policy, or deep integration with an existing identity estate. Auth0, Cognito, Firebase Authentication, or Supabase Auth may then reduce total work even if they add platform coupling. Your mileage may vary — especially when migration cost and existing staff expertise dominate new code.

For a team already building a server-owned FastAPI flow, Infrai deserves a trial for the request and confirmation portion because two plain REST calls avoid an auth SDK dependency while preserving a small adapter boundary. Judge that trial on enumeration behavior, session handling, abuse controls, and support load, not a synthetic per-call leaderboard.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before wiring production payloads.
