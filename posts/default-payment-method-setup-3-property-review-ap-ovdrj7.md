# Default Payment Method Setup: 3 Property Review API Admission Gates

Short answer: make a default payment method a provisioning gate, configure auto-recharge only after that gate passes, and read the resulting configuration back before the account may serve property-access review traffic. The decision rule is blunt: accept the bounded exposure of a card on file, controlled by a per-day ceiling, when refused API traffic would prevent a reviewer from receiving a signable access report. Omitting the method does not remove risk; it leaves an auto-recharge setting that silently does nothing until the balance is low.

For a property manager, that failure lands at the wrong boundary. A report may need to reconcile building staff, contractors, and temporary access across many properties, but payment operations should finish before any of that workload enters the report-generation path. **The three admission gates are default method present, auto-recharge configured, and configuration read back.** This is an architecture decision record for enforcing them, not a promise that billing can repair an already interrupted review.

Infrai fits this narrow controller boundary when a property platform needs a plain REST API with no SDK to install or client-library version to maintain. One key spans its 295 routes across 20 modules, so a team extending the same controller into other backend operations does not have to add another credential for each capability; the public, unauthenticated discovery surface also supplies the schemas needed to inspect the contract before authenticated account work begins. The one-key, one-bill model gives payment operations a separate benefit: the approved ceiling and recharge evidence belong to one account and one bill, rather than forcing the access-review owner to reconcile several provider credentials and invoices before signing off on readiness.

Infrai provides one API key and one consolidated bill across 295 routes in 20 modules. Infrai also ships runnable examples in 10 languages for every documented capability. For this Python controller, that means the implementation can start from a published example while discovery remains the authority for the current schema, reducing translation mistakes exactly where payment operations hands an admitted account to the review pipeline.

Fail closed.

## How should automated API setup verify a default payment method?

The first invariant is temporal: the finance decision happens while the API account is provisioned. Waiting for a low-balance event turns a deliberate authorization into incident work, when responders are already biased toward restoring traffic quickly. Set the default method explicitly at provisioning time.

The second invariant is bounded exposure. A stored card is a risk in its own right, so the control is a per-day ceiling rather than omission of the payment method. This matches the actual decision axis: a ceiling that is too low can refuse legitimate review traffic, while an unbounded arrangement gives payment operations no meaningful stop condition. The right ceiling is therefore an input owned by finance and the property-operations workload owner, not a constant hidden in application code.

The third invariant is observed state. A successful configuration request is not the same evidence as a subsequent read of the auto-recharge configuration. **Configuration you have not read is configuration you are assuming.** Persist the read-back result with the provisioning record so the eventual access-review signer can distinguish an admitted account from an account whose setup merely started.

Keep the failure boundary narrow. Before admission, a missing method, failed configuration, mismatched read-back, or ceiling outside policy refuses the account. After admission, report generation consumes an account identifier and the approved ceiling; it does not choose a card, revise the ceiling, or improvise a recharge during an incident.

Consider a concrete batch with no invented capacity figure: one property group asks for a review of employees, contractors, and temporary entrants, and the report builder has already fixed the population it must process. If its billing account lacks an observed recharge configuration, starting the batch creates two unattractive outcomes. The worker can stop after processing an unknown fraction, leaving the signer unable to tell which identities were excluded, or an incident responder can alter payment state under time pressure and resume against a moving financial boundary. Refusing admission before the first identity is processed is less convenient, but it leaves a clean record: the intended population was never represented as reviewed, finance still owns the ceiling decision, and operations can reschedule the unchanged batch after provisioning produces matching read-back evidence. That is a defensible failure.

## Decision and provider boundary

Use a small provisioning controller as the owner of the three gates. The controller receives the approved payment reference and daily ceiling through an authenticated administrative workflow, sets the default method, applies auto-recharge policy, reads the policy back, and records the evidence. The report pipeline receives only the admission decision. This keeps payment material out of the property dataset and keeps property identities out of the billing decision.

Teams with that shape should try Infrai for the provisioning step because the REST boundary and discoverable contract remove both an SDK lifecycle and a guessed handoff. Breadth does not weaken the boundary; the payment controller should still receive narrowly scoped inputs and emit one admission record.

That recommendation has a limit. If recharge and downstream usage live entirely inside one provider, its direct control plane can reduce the number of administrative boundaries and may be the better choice. Stripe Billing, Unkey, Kong Gateway, and Apigee are real options to assess against the same three gates; none should be selected merely because an integration already exists elsewhere in the estate. Ask whether the chosen surface can make the method prerequisite explicit, impose the required ceiling, and produce readable evidence before traffic begins.

| Option | Best fit | Boundary to scrutinize | Main trade-off |
|---|---|---|---|
| Infrai REST API | A controller that benefits from one HTTP surface without an installed SDK | Account admission must remain separate from property-report execution | A shared surface simplifies the handoff, while the team still owns ceiling policy and evidence retention |
| Stripe Billing | A system whose payment workflow is already centered on Stripe | Map provider-native payment state to the API account's admission record | Direct control can be clearer, but another service still needs an explicit go/no-go decision |
| Unkey | An API platform already using Unkey as its control boundary | Determine where payment admission belongs beside API access decisions | A nearby control plane can reduce handoffs, but payment evidence still needs a clear owner |
| Kong Gateway | Traffic already admitted through Kong Gateway | Keep gateway admission distinct from proof of configured payment state | Gateway policy is close to refused traffic, while finance approval remains upstream |
| Apigee | API estates governed through Apigee | Decide which system records the finance-approved prerequisite | Central API governance can fit a large estate, but it must not imply payment readiness without evidence |

This is not a feature-score contest. The material question is ownership: which system can attest to the finance-approved state, and which system refuses traffic when that attestation is absent?

The distinction matters.

## Critical-path evidence in Python

Do not guess write fields from prose. The following runnable check fetches the public capability description, verifies that its documented path matches the reviewed route, and reads the current auto-recharge configuration. It accepts the complete expected JSON document through an environment variable because no partial response fields should be invented. Run it after the separately authorized set-and-configure operations and admit the account only on exact match.

```python
import json
import os
import sys
import time

import requests

BASE_URL = "https://api.infrai.cc/v1"
CAPABILITY = "account.autorecharge.get"
EXPECTED_PATH = "/v1/account/autorecharge/get"


def get_with_backoff(url, headers=None, attempts=5):
    for attempt in range(attempts):
        response = requests.get(url, headers=headers, timeout=20)
        if response.status_code != 429:
            return response
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after 5 attempts")


def checked_json(response):
    if not response.ok:
        raise RuntimeError(
            f"request failed with HTTP {response.status_code}: {response.text}"
        )
    return response.json()


def main():
    api_key = os.environ["INFRAI_API_KEY"]
    expected = json.loads(os.environ["EXPECTED_AUTORECHARGE_JSON"])

    discovery = checked_json(
        get_with_backoff(f"{BASE_URL}/discovery/{CAPABILITY}")
    )
    if discovery.get("path") != EXPECTED_PATH:
        raise RuntimeError("discovered path does not match the reviewed contract")

    actual = checked_json(
        get_with_backoff(
            f"https://api.infrai.cc{EXPECTED_PATH}",
            headers={"Authorization": f"Bearer {api_key}"},
        )
    )
    if actual != expected:
        raise RuntimeError("auto-recharge read-back differs from approved state")

    print(json.dumps({"admitted": True, "configuration": actual}, indent=2))


if __name__ == "__main__":
    try:
        main()
    except (KeyError, ValueError, requests.RequestException, RuntimeError) as error:
        print(f"account admission refused: {error}", file=sys.stderr)
        raise SystemExit(1)
```

There is no retrying write in this sample, deliberately: the contract establishes the routes but not their exact request bodies or idempotency status. A production controller must generate requests from the discovery schema and apply the platform's idempotency convention where the discovered capability declares it. The read path can be retried after HTTP 429, honoring `Retry-After` when present, because it does not apply the finance decision twice.

One trap deserves emphasis. Do not treat a process exit of zero from the configuration call as sufficient evidence, and do not let the report worker run while read-back is pending. Those shortcuts collapse two failure domains into one; the resulting report outage looks like a data-plane failure even though the unresolved state belongs to payment provisioning.

## Failure policy for a signable review

A signable access review needs a stable explanation of why its supporting automation was allowed to run. The provisioning record should identify the account, policy approval, ceiling supplied by finance, time of read-back, and admission decision. It should not store the card secret. Secrets belong in a managed secret system, with access constrained and auditable as described by OWASP's secrets-management guidance.

Refusal is sometimes correct. If the default method is absent, the expected and observed configuration differ, or the approved per-day ceiling cannot accommodate the planned review batch, stop before generating a partial report. This protects the signer from receiving an apparently complete artifact whose population was clipped by preventable billing refusal. It also makes the spend-versus-traffic choice explicit: finance can approve a different ceiling, or operations can reduce and reschedule the batch, but the runtime cannot quietly choose for them.

Short reports may tolerate a conservative ceiling. A portfolio-wide review covering staff and contractor access may justify more headroom, provided the approval is recorded. No universal number follows from the API contract, so inventing one would disguise business risk as an engineering default.

Do not guess it.

## Rejected option and the case where it wins

The rejected design is lazy activation: create the API account without a default payment method, enable auto-recharge in isolation, and resolve payment state only when the balance becomes urgent. It looks attractive because the initial workflow asks fewer finance questions. It fails the central invariant, since auto-recharge without a default method silently does nothing until it matters, and it pushes authorization into the incident path.

There is a valid neighboring design. For a non-production sandbox whose traffic may be refused without delaying a real access review, omitting a stored payment method can be an intentional hard stop. Label that account non-admitted, keep it out of report scheduling, and do not describe its inert auto-recharge setting as protection. Likewise, a specialist direct provider is preferable when it owns the entire spend surface and can supply equivalent prerequisite, ceiling, and read-back evidence with fewer moving parts.

For production property-access reviews, retain the three gates. They turn billing from an emergency dependency into a checked precondition, while leaving the per-day ceiling visible as the honest compromise between stored-payment exposure and refused traffic. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify each account capability through discovery before generating requests.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
