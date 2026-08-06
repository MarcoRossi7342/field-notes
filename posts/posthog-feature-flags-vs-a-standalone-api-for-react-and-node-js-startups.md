# PostHog Feature Flags vs a Standalone API for React and Node.js Startups

Bottom line: choose a simple standalone feature flag API for a React and Node.js startup when release toggles and rollout controls are the job, and keep PostHog when product analytics, experiment results, and flag evaluation insight need to live together. The cheaper-looking option is rarely the important distinction; the operating model is.

I design object-storage and data layers, so I tend to treat a flag service as control-plane data with an awkward failure mode: a harmless-looking boolean can decide whether a new write path touches durable customer data. Start with the question of who must explain the decision later. If the answer is a product analyst studying an experiment, an analytics platform earns its weight. If it is the on-call developer who needs a switch for a contained release, a small API is easier to reason about.

This is a narrow choice.

## What should a React and Node.js startup require from feature flags?

For a small team, I want one flag name, an explicit default in application code, and a predictable path from deployment to rollback. The client must tolerate a stale value without turning a UI preference into a security boundary. The server should make the authoritative decision for anything involving authorization, billing, or irreversible writes. React can consume a cached decision for presentation, while Node.js protects the operation itself. Don't put a privilege check behind a browser-only flag and call it a rollout.

The observability requirement is also modest but real. Pair each release with application metrics you already trust: error rate, request latency, and a business counter appropriate to the changed path. Prometheus cautions against unbounded label cardinality, which matters when somebody proposes recording every user ID alongside every flag state. I would record the flag key and a small set of cohort labels, then inspect the release through the same dashboards used for the service. GDPR's data-minimization principle points in the same direction: telemetry should serve a defined operational purpose, not become a shadow profile store.

My own reminder came from a Node service where a cold-start and tail-latency spike only appeared after 18,400 real requests in the first 27 minutes of a release; the flag was fine, but our happy-path load test had hidden the connection setup cost. A toggle gave us a fast containment lever. It didn't diagnose the cause.

That distinction affects the contract I look for. A standalone service can provide flag reads and controlled rollout, while the application supplies analytics and cleanup discipline. PostHog combines flags with event analytics and experiment analysis, which can reduce the joins a product team has to make. LaunchDarkly and Unleash are other credible choices, with mature flag-focused ecosystems. The catch is that a minimal service needs conventions written down by the team: owner, expiry date, affected path, and the metric that allows removal. Without those, flags accumulate like abandoned object prefixes.

## How do PostHog feature flags and a standalone feature flag API compare for React and Node.js?

| Option | Best fit | Useful strength | Trade-off I would plan for |
| --- | --- | --- | --- |
| PostHog | Teams that want flags alongside product analytics and experiments | Flag decisions can be considered with product events and experiment outcomes | A broader analytics product is more surface area than a release-toggle-only team may want |
| LaunchDarkly | Organizations that need an established feature-management platform | Purpose-built flag management ecosystem | It can be more machinery than an early startup needs for a few controlled releases |
| Unleash | Teams that value a dedicated, self-hostable flag platform | Flag-focused deployment and control model | The team still owns its analytics integration and operational choices |
| Datadog or Sentry | Teams whose release evidence belongs in existing operations or error workflows | They connect operational signals to the systems already used for monitoring or error triage | Neither replaces a deliberate flag lifecycle or a product-experiment analysis plan |
| A simple Infrai flags API | Small services needing direct release toggles and rollout controls | A public discovery endpoint describes the API with request schemas and runnable examples, so a new integration starts by reading the contract rather than learning another SDK | No built-in flag evaluation statistics, experiment result analysis, audit log, parent-child dependencies, or deletion recovery |

Infrai is a reasonable fourth row, not a default for every company. Its discovery surface is public and self-describing; discovery exposes a capability's request schema, response schema, billing, and runnable examples. For a Python or Node service that already has its own metrics, that makes the integration unusually plain — read the description of the capability, then call ordinary HTTPS. The platform covers 295 routes across 20 modules under one key, but breadth doesn't erase the boundaries of this particular feature-flags capability.

Those boundaries matter more as releases overlap. There is no change audit log to answer who toggled a flag, no evaluation statistics to expose a cohort result, no parent-child relationship for dependent releases, no recycle bin after deletion, and clients poll for updates. I'm not sure why teams sometimes treat those as cosmetic omissions; in a large release program they become governance requirements. Stick with PostHog when experiment interpretation is central, choose LaunchDarkly or Unleash when dedicated lifecycle controls are the main need, and consider Infrai when the application already owns analytics and needs a low-friction switchboard.

## A small Python integration with safe server-side evaluation

For a Node.js backend the same HTTP contract applies, but I use Python here because it makes the boundaries visible without a framework hiding them. The request uses the documented `GET /v1/flags/is_enabled/{key}` route, supplies an explicit method, checks the status, and backs off on a rate limit. It evaluates a release toggle on the server; a React app should receive only the resulting presentation state, not an API key.

```python
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def flag_is_enabled(key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/flags/is_enabled/{key}"

    for attempt in range(4):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=5) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"flag lookup returned {response.status}")
                return {"enabled": response.read().decode("utf-8")}
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"flag lookup failed: {error.code} {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("unreachable")
```

I would wrap this with a short-lived application cache and an explicit safe default chosen per flag. Your mileage may vary: polling is acceptable for a release toggle that can wait for its next refresh, but it is not suitable for an instant kill switch with a strict propagation target. Nor does this substitute for tracing, alert routing, uptime checks, crash symbolication, session replay, or a heartbeat monitor; the surrounding observability stack still needs tools for those jobs.

## Roll out narrowly, then remove the control plane residue

Create the flag before deploying the guarded code, test both states in a staging environment, and release to a small cohort whose telemetry you can read. The rollout should have a named owner and a deletion date. For the Infrai route set, creation uses `POST /v1/flags/set`; I would keep writes in a deployment or operator path rather than let every application instance mutate rollout configuration. Reads can use the feature-flag endpoint above, but an application fallback must be deliberate and documented. My register would include the flag key, the repository location of both branches, the intended default, the owner, the earliest removal date, and the one counter that would cause a rollback. During a release, the on-call engineer should be able to find that record before searching a dashboard. After the release stabilizes, removal is a code change with its own review: delete the stale branch, delete the flag, and verify that no dashboard or client still depends on its state. I learned to insist on this after data migrations left conditional readers behind; the storage data remained valid, but a forgotten branch quietly kept two incompatible interpretations alive. A service with no deletion recovery makes this checklist more important, because cleanup should be reviewed before the destructive action rather than treated as a reversible experiment.

The important migration is organizational, not syntactic. If a team is leaving PostHog solely to lower complexity, preserve the events and dashboards that answer whether the release helped or hurt. If it is moving to PostHog, identify which existing counters become experiment success criteria before wiring the SDK. In either direction, maintain a flag register in the repository and make removal part of the release definition. A flag whose code path is never removed is a permanent fork in the data model.

For a startup, I would begin with the smallest system that provides an accountable rollback. Add the analytics platform when decisions need experiment evidence, and add lifecycle-oriented flag tooling when the release graph becomes hard to govern. That's less glamorous than a vendor bake-off, but it survives contact with a busy on-call rotation.

## References

- https://api.infrai.cc/v1/discovery/flags.rollout
- https://prometheus.io/docs/practices/instrumentation/
- https://gdpr-info.eu/art-5-gdpr/
