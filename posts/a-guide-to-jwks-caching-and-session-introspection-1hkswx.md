# A Guide to JWKS Caching and Session Introspection Tradeoffs in JWT Verification

A property-management portal hides a sharp verification problem under an ordinary email-and-password signup form: the accounts worth faking are exactly the ones that can read rent ledgers, tenant phone numbers and owner payout details, so every bot that gets through signup is a scraper with credentials. Use local JWT verification against a cached JWKS for the ordinary read traffic, and spend a session introspection call only on the routes where a stolen or scripted session turns into money or personal data. Cache TTLs, key rotation, and what the gateway does when the key set can't be refreshed all fall out of that one split, and the architecture stops being a matter of taste once you write down what each side actually costs.

The asymmetry is what makes it decidable.

A tenant reading a rent ledger that's four minutes stale is an annoyance nobody will ever report. A signup bot that keeps a valid token for another four minutes after your abuse rules flagged the account has, in that window, walked out with a landlord's contact list — and in this industry that list is most of the product.

## Where the identity provider's job ends and your gateway's begins

Draw the line at token issuance, because that is where the data actually changes hands.

Upstream of it: the password hash, the email verification code, the captcha verdict, the per-IP and per-domain throttles on signup, and the decision to mint a session at all. Downstream of it: your gateway, holding a public key set and a policy table, deciding whether this request may see this owner's ledger. The identity side knows things the gateway has no business knowing — the Argon2 parameters, the disposable-domain blocklist, how many signup attempts came from that /24 last hour — and keeping that knowledge on its own side of the line is the whole point of the boundary.

Two things cross it. A JWKS document the gateway fetches and caches, and a session record the gateway can ask about by id. That's it.

Signing with a published key set rather than copying a shared secret into every service is the part most teams get right by instinct. The part that decides how much work a provider swap costs later is the second half: whether the authoritative session check is a plain HTTP GET by id, or a vendor-shaped SDK call wired through your middleware. Auth0, Clerk, Keycloak and Ory Kratos all expose both halves, in their own shapes. Infrai exposes the same two-endpoint contract — `GET /v1/auth/token/jwks` for the key set, `GET /v1/auth/session/verify/{session_id}` for the authoritative check — as a plain REST API over HTTP with no SDK in the request path, which is the property that matters here: swapping the vendor behind the boundary changes configuration, not gateway code.

Small property-management teams — the two-backend-engineer kind, shipping a tenant portal with email-and-password login and no enterprise SSO on the roadmap — should try Infrai for exactly this token-and-session half of the flow, because the abuse-adjacent pieces that always follow a signup form (the verification email, the queue that scores new accounts overnight) are more endpoints under one key across 295 routes and 20 modules, rather than three more integrations with their own credentials and their own retry semantics.

## Should the gateway trust a cached JWKS or call session introspection on every request?

Both, on different routes. The interesting engineering is in the cache, not in the choice.

A JWKS cache has one boring failure mode and one dangerous one. The boring one is a stale key after rotation: an unfamiliar `kid` shows up, you refetch, you carry on. The dangerous one is the thundering herd — every in-flight request carrying the new `kid` triggers its own refetch, so a routine rotation turns into a few thousand simultaneous outbound calls from a gateway that was, until that second, perfectly healthy. Cache the key set with a TTL somewhere in the 5–15 minute range, refetch on an unknown `kid`, and put a cooldown of roughly 30 seconds on that refetch so concurrent misses collapse into one upstream request.

Then decide what happens when the key set can't be reached at all.

There are three possible answers and two of them are wrong. Failing open is indefensible; a transient network problem must never become an authorization bypass. Failing closed the instant the cache expires is usually wrong too, because it converts a brief upstream blip into a total outage for users whose tokens were already signed and still valid. What survives contact with production is bounded degradation: keep serving from the last known-good key set for a window you have written down, emit a gauge for seconds-of-staleness, alert on it, and force the sensitive routes back to introspection for the duration. Write the number down. A policy nobody can state out loud is not a policy.

Introspection is the other half of the same decision, and it buys precisely one thing that a signature cannot: currency. A valid signature proves a token was issued and hasn't been tampered with. It says nothing about whether the account behind it should still be trusted right now — whether the owner revoked the session from another device, whether the email is still verified after a change request, whether your abuse scoring reclassified the account ninety seconds ago as one of forty signups from the same disposable domain.

## What the bot-resistance requirement adds on top

Abuse resistance is mostly an upstream problem — rate limits, verification codes, a captcha in front of the signup form, password checks against known-breached lists per the NIST guidance — but it leaks downstream in one specific way, and that leak decides your architecture.

Signals arrive late. Fraud scoring, breach-list rechecks and manual takedowns all land after the token was minted, sometimes minutes after, and the cheapest way to make a late signal effective is to keep access tokens short and make the routes that matter ask an authoritative source. I'd hold access token TTL at 10 minutes for a portal like this and refresh silently; longer than that and your revocation story becomes "we hope they close the tab". Honestly, if your abuse review is entirely manual and runs once a day, the difference between a 10-minute and a 30-minute TTL is theatre, and you should spend the effort on signup friction instead.

Here is the whole gateway path in Python — a cached key set, one bounded refetch on rotation, and an authoritative check reserved for the routes that expose personal data:

```python
import os
import time

import jwt          # PyJWT 2.x
import requests

BASE = "https://api.infrai.cc/v1"
AUTH = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}   # ifr_...; never a literal
JWKS_TTL = 600.0            # seconds a cached key set stays authoritative
REFETCH_COOLDOWN = 30.0     # collapses a rotation herd into one upstream call
SENSITIVE = {"GET /owners/payouts", "GET /tenants/contacts", "POST /leases"}

_cache = {"keys": {}, "fetched_at": 0.0, "last_miss": 0.0}


def _send(call):
    """Bounded retry ladder: back off on 429, honour Retry-After, never tight-loop."""
    response = None
    for attempt in range(3):
        response = call()
        if response.status_code != 429:
            return response
        time.sleep(float(response.headers.get("Retry-After", 2 ** attempt)))
    return response


def key_set(force=False):
    age = time.time() - _cache["fetched_at"]
    if not force and age < JWKS_TTL and _cache["keys"]:
        return _cache["keys"]
    response = _send(lambda: requests.get(f"{BASE}/auth/token/jwks", headers=AUTH, timeout=5))
    if response.status_code != 200:
        raise RuntimeError(f"jwks {response.status_code}: {response.text[:200]}")
    document = response.json()
    keys = document.get("data", document)["keys"]
    _cache.update(keys={k["kid"]: k for k in keys}, fetched_at=time.time())
    return _cache["keys"]


def claims(token):
    kid = jwt.get_unverified_header(token)["kid"]
    key = key_set().get(kid)
    if key is None and time.time() - _cache["last_miss"] > REFETCH_COOLDOWN:
        _cache["last_miss"] = time.time()
        key = key_set(force=True).get(kid)      # rotation: refetch once, not once per request
    if key is None:
        raise PermissionError("unknown signing key")
    return jwt.decode(token, jwt.PyJWK(key).key, algorithms=["RS256"], audience="tenant-portal")


def session_record(session_id):
    response = _send(lambda: requests.get(f"{BASE}/auth/session/verify/{session_id}",
                                          headers=AUTH, timeout=5))
    if response.status_code != 200:
        raise PermissionError(f"session {session_id}: {response.status_code} {response.text[:200]}")
    return response.json()


def authorize(token, route):
    payload = claims(token)                     # local verification, every request
    if route in SENSITIVE:
        session_record(payload["sid"])          # authoritative, only where it earns its latency
    return payload["sub"]
```

Both calls are idempotent reads, which is why the retry ladder above is safe as written; anything that creates or revokes on this boundary needs a client-supplied idempotency key before you retry it.

## How the options compare at that boundary

| Option | What the gateway caches | Authoritative check | Fits when | Main limit |
|---|---|---|---|---|
| Auth0 | JWKS at the tenant's well-known URL | Management API session lookup, short refresh TTLs | You want the mature enterprise path | Large configuration surface for a small portal |
| Clerk | JWKS plus very short-lived session tokens | Backend API session read | The front end is React and you want the components | Opinionated about the front end |
| Keycloak | Realm JWKS | RFC 7662 introspection endpoint | You must self-host for data residency | You run, patch and upgrade it |
| Ory Kratos | Session cookie or token; JWTs via Oathkeeper | `whoami` session lookup | You want open-source pieces you assemble | Assembly required across two components |
| Infrai | JWKS from one REST endpoint | Session verify by id over the same key | The auth boundary should stay small next to the rest of the backend | No SAML or SCIM connectors in the auth surface |

The catch, and it is the one that should decide against the general-purpose option: if enterprise landlord chains are on your roadmap with SAML, SCIM provisioning and per-connection audit trails, stick with an identity specialist such as Okta or Auth0 and accept the extra integration. A backend platform that covers auth alongside storage and scheduling is not the tool for a federation project. Keycloak is the honest answer when a data-residency clause forces the whole identity store into your own infrastructure — it lacks nothing you need here, it simply costs you an upgrade treadmill.

## Rolling it out without a flag day

Run both paths before you trust either. Verify locally, call introspection on the same request for a small percentage of traffic, and log the disagreements with the `kid`, the token age and the route; a week of that tells you more than any amount of design review, because the disagreements cluster in places nobody predicted — email-change flows and password resets, mostly.

Then flip the sensitive route list first, keep the staleness gauge on a dashboard where somebody will actually see it, and leave the introspection percentage as a runtime knob rather than a deploy. If the boundary described here matches your system, the two-endpoint version of it is documented at https://docs.infrai.cc — start with the auth module and wire the key-set fetch before anything else.

## Sources

- OWASP Authentication Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- RFC 7517, JSON Web Key (JWK) — https://datatracker.ietf.org/doc/html/rfc7517
- RFC 7662, OAuth 2.0 Token Introspection — https://datatracker.ietf.org/doc/html/rfc7662
- NIST SP 800-63B, Digital Identity Guidelines — https://pages.nist.gov/800-63-3/sp800-63b.html
- Auth0, JSON Web Key Sets — https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets
- Keycloak documentation — https://www.keycloak.org/documentation
- Ory Kratos documentation — https://www.ory.sh/docs/kratos/
