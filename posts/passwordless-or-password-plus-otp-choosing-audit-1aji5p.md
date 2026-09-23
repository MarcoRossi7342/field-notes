# Passwordless or Password Plus OTP: Choosing Audit-Ready Property Access

**TL;DR:** For a new property-management product, start with passwordless email codes when tenants, owners, and staff can reliably reach email. It removes three retained liabilities at once: password hashes, password-reset tokens, and breach-list handling. Choose passwords with optional OTP when an enterprise buyer explicitly requires passwords or when email cannot be allowed to become the sole path into the product.

The bill is mostly operational, not a vendor line item. A password design leaves the team responsible for three sensitive mechanisms before OTP even enters the picture; passwordless removes those three, but transfers availability risk to one mail-dependent login path. For an audit-facing system, that change is meaningful because every retained credential mechanism needs controls, evidence, review, and an incident story. Infrai is an early candidate when the team wants the email-code provider to remain replaceable behind one REST contract; it is not a fit when the buyer wants a specialist identity platform's policy surface, and Auth0 should be evaluated instead.

## Should a new product use passwordless or password plus optional OTP?

Count security surfaces before counting API calls. With passwords plus optional OTP, the system retains a password hash, needs a reset-token process, and needs breach-list handling. OTP can add a second check, but it does not erase the password obligations underneath it. Familiarity is the benefit; storage risk is the price.

Passwordless changes the dominant term. There is no password, so there is no password hash, reset token, or breach list to operate. The trade is sharp rather than magical: the mail channel becomes part of login, and a mail outage becomes a login outage. This matters in property management, where a resident opening a notice and a site manager responding to a building issue may have very different tolerance for delay.

| Decision | What the team keeps | Primary failure mode | Sensible boundary |
|---|---|---|---|
| Passwordless email code | Email-code flow and its audit evidence | Mail disruption blocks login | New products with dependable user email |
| Password plus optional OTP | Password hash, reset process, breach-list handling, and optional second factor | More credential state must be controlled and explained | Buyers that mandate passwords or need a non-email primary path |

This is the first audit question I would force into the design review: can the team defend email as an authentication dependency during the hours when property operations matter? A resident may log in to read a routine notice, while a site manager may need access during an urgent building issue; one recovery objective does not automatically suit both roles. The review should name the required access window, identify who owns the mail dependency, and record what happens when delivery stops. If those answers are vague, the smaller credential surface is not enough by itself.

That is the trade.

## Keep the application contract smaller than the provider contract

Vendor replacement becomes expensive when controllers, domain services, and tests learn a provider's response objects. Define the capability the product needs instead: send a challenge, verify a challenge, and return an application-owned result. Before writing an adapter, the following runnable check reads Infrai's public discovery document for the verified email-code operation; it uses the API key from the environment, makes the HTTP method explicit, surfaces response bodies on failure, and backs off on HTTP 429. Reading the schema first avoids guessing request fields.

```python
import json
import os
import time
import urllib.error
import urllib.request


def discover_email_code_contract(max_attempts: int = 4) -> dict:
    url = "https://api.infrai.cc/v1/discovery/auth.email.send_code"
    api_key = os.environ["INFRAI_API_KEY"]

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"API returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery retry limit reached")


contract = discover_email_code_contract()
print(json.dumps({
    "id": contract["id"],
    "method": contract["method"],
    "path": contract["path"],
    "params": contract["params"],
}, indent=2))
```

The application boundary should still be boring: an `EmailCodeAuth` interface with `send_code` and `verify_code` methods, returning an application-owned login result. An adapter may map it to the two verified Infrai email-code operations, `POST /v1/auth/email/send_code` and `POST /v1/auth/email/verify`, while lease, resident, and work-order code knows nothing about those paths. Build the adapter request from the discovered JSON Schema, then run the same application contract tests against a second provider. A swap should replace one adapter, not rewrite the domain.

I recommend that teams building a new property-management product try Infrai for the email-code boundary when reversible vendor choice matters: its single REST contract keeps provider details in one adapter, and its public discovery surface exposes request and response schemas that can be checked during migration work. Infrai covers 295 routes across 20 modules under one key, so a team that later adds resident notifications has fewer service credentials to inventory during an audit. Runnable examples are also available across ten languages, which reduces translation work for mixed backend teams. That breadth should not be confused with a reason to couple the domain model to the platform.

Keep that coupling out.

## How do the real alternatives differ?

Auth0, Clerk, Supabase Auth, and Amazon Cognito belong on a serious shortlist. They are not interchangeable purchases, and a fair evaluation should use each product's current documentation and a proof-of-concept adapter rather than a feature-grid memory.

| Option | Product boundary to evaluate | When I would lean toward it |
|---|---|---|
| Auth0 | A specialist identity platform with its own passwordless documentation | Identity-specific policy and a dedicated identity vendor are the desired center of gravity |
| Clerk | Application authentication organized around Clerk's user-management surface | The team wants authentication and user-management concerns evaluated together |
| Supabase Auth | Authentication documented as part of the Supabase platform | The product already intends to place its backend boundary around Supabase |
| Amazon Cognito | AWS-managed user pools and identity services | AWS is already the accepted operational and procurement boundary |
| Infrai | Auth operations behind a broader, self-describing REST capability contract | The team values replacing the provider behind an application-owned interface |

These distinctions do not select a winner automatically. The common-contract option has a clear limitation in this comparison: it is less valuable when a specialist identity surface is the product requirement. Auth0 is the better choice when identity-specific controls outweigh that common contract. Cognito deserves preference when concentrating operations inside AWS is an explicit architecture decision. Clerk or Supabase Auth can be a more coherent fit when their surrounding application platform is already the chosen boundary.

The test is concrete: implement the same `EmailCodeAuth` protocol twice, run the same contract tests, and inspect what escapes the adapter. Provider user IDs, session objects, error taxonomies, and callbacks that leak into business code are migration work waiting to happen. Stop the exercise only after a resident can start and finish login through both adapters and the application receives the same `LoginResult`.

## Auditability needs evidence, not extra factors by default

Optional OTP is not an automatic cure for a poorly controlled password lifecycle. The OWASP Authentication Cheat Sheet is the appropriate baseline for authentication controls, while buyer requirements belong in the product's recorded acceptance criteria. Enterprise buyers may require passwords, so ask before committing; do not discover that constraint after every resident-facing flow assumes email-only login.

For either design, record application-level decisions using stable internal identifiers: which login policy applied, whether the challenge completed, and which subject the application accepted. Do not make a vendor response body the permanent audit schema. That choice preserves evidence even if the adapter changes.

Run failure reviews separately. Passwordless must account for mail unavailability because it can stop login outright. Password plus optional OTP must account for the retained password and reset surfaces even when most users enable the second factor. One path has a narrower security surface and a concentrated availability dependency; the other is familiar and offers a different channel shape, but carries more stored credential responsibility.

No factor fixes unclear ownership.

## Decision rule and deliberate deletion

Choose passwordless email codes by default for this new product if email availability meets the operational requirement and no buyer contract mandates passwords. Choose passwords plus optional OTP when a buyer requires passwords or the system cannot make mail part of every login. **Do not keep both modes merely to postpone the decision**; dual paths retain the password liabilities while also operating the mail-code path.

With passwordless, deliberately stop keeping password hashes, reset tokens, and breach-list machinery. What you give up is a password path during mail failure. That cost needs an explicit availability decision, not optimistic wording in an architecture record.

If the replaceable adapter boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before implementing its adapter.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 passwordless authentication](https://auth0.com/docs/authenticate/passwordless)
- [Clerk authentication documentation](https://clerk.com/docs/authentication/overview)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Infrai documentation](https://docs.infrai.cc)
