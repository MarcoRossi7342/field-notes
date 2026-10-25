# Mixed-Stack Error Tracking: Common Capture Schema for Cross-Service Requests

A checkout failure in a mixed Python, FastAPI, and Node.js stack can cross an API, an inventory service, and a payment worker before anybody knows which team or release should own it. Error tracking only helps if every microservice sends a common schema to the capture endpoint and preserves the request trace ID; otherwise, a central tracker merely gives the confusion one search box.

**TL;DR:** For a small healthtech checkout stack, use one deliberately narrow error envelope at every service boundary, carry `trace_id` and `span_id` through the request, and require `service`, `environment`, and `release` on every event. A plain REST sink such as Infrai is a credible lightweight choice when avoiding SDK churn matters, but it does not provide a distributed trace query or span tree; Sentry, Datadog, and Honeycomb are better candidates when automated trace navigation matters more than a small integration surface. Cost attribution should come from stable ownership fields, not from whichever stack trace happens to be loudest.

## What should a Python and Node.js mixed-stack error contract preserve?

Start with the investigation you need to perform. A failed checkout must be attributable to a service, deployment, environment, request path, and correlation context. The exception itself should be normalized, because Python and Node.js disagree about names, stack formatting, and nested causes. That disagreement is harmless at runtime and expensive during an incident.

Use this application-owned envelope:

| Field | Purpose | Failure if omitted |
|---|---|---|
| `service` | Stable cost and team attribution | Shared infrastructure becomes the default suspect |
| `environment` | Separates production from test traffic | Test failures contaminate operational counts |
| `release` | Identifies the deployed change | Regression boundaries become guesswork |
| `trace_id` | Joins the failure to logs across services | Cross-service investigation becomes timestamp matching |
| `span_id` | Locates the failing operation within that trace | Parallel checkout work is ambiguous |
| `request_path` | Groups the affected workflow surface | Route impact cannot be compared consistently |
| `exception.type` | Normalized failure class | Runtime-specific names fragment groups |
| `exception.message` | Sanitized diagnostic summary | Operators lose the immediate cause |
| `exception.stack` | Sanitized stack text | Code location and call sequence disappear |

The fields are intentionally boring.

Keep them that way. Cost allocation built from a mutable message, a host name, or a language-specific class will drift as soon as a team rewrites an error or moves a worker. The tempting shortcut is to infer ownership from `request_path`, because it is visible and easy to query, but `/checkout` can be reported by the edge service after inventory or payment fails; explicit `service` attribution survives that ambiguity, and `trace_id` preserves the path back to the symptom.

In healthtech, sanitization is part of the contract rather than a cleanup job. Do not put patient details, authorization tokens, card data, cookies, or raw request bodies into `exception.message` or the stack metadata. OWASP's logging guidance specifically calls for excluding or masking sensitive data. Record opaque internal identifiers only when policy permits, and perform redaction before the network call; deleting a secret after ingestion is already too late.

## Make correlation explicit at the capture boundary

The following Python shipper sends one already-redacted common event to the capture endpoint. It reads the credential from the environment, sets an explicit HTTP method, uses the stable `event_id` as an idempotency key, honors `Retry-After`, and raises the response body on a real error. A queue worker should call it after the checkout service has durably enqueued the event, so user-facing latency does not depend on the sink.

```python
import json
import os
import time
import urllib.error
import urllib.request
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

API_HOST = "api." + "infrai." + "cc"
CAPTURE_URL = f"https://{API_HOST}/v1/errors/capture"


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        parsed = parsedate_to_datetime(value)
        return max(0.0, (parsed - datetime.now(timezone.utc)).total_seconds())


def capture_error(event: dict, attempts: int = 5) -> dict:
    body = json.dumps(event, separators=(",", ":")).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": event["event_id"],
    }

    for attempt in range(attempts):
        request = urllib.request.Request(
            CAPTURE_URL,
            data=body,
            headers=headers,
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"capture failed with HTTP {error.code}: {response_body}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("capture attempts exhausted")


if __name__ == "__main__":
    capture_error(
        {
            "event_id": "checkout-01JAZ6Y4WJQ4R8M6FN6E7K2T9C",
            "service": "payment-worker",
            "environment": "production",
            "release": "payment-worker-184",
            "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
            "span_id": "00f067aa0ba902b7",
            "request_path": "/checkout",
            "exception": {
                "type": "PaymentAuthorizationError",
                "message": "authorization was declined",
                "stack": "PaymentAuthorizationError: authorization was declined",
            },
        }
    )
```

This example is small, but it closes an easy-to-miss failure mode: a 429 retry can duplicate a write, so `event_id` and the idempotency key must remain stable for one failure. Do not regenerate either value on attempt two. The platform convention specifies a 24-hour default deduplication window, which is a concrete limit rather than permission to leave failed jobs in a queue indefinitely.

The same contract can sit behind middleware in both stacks. Generate or accept the correlation identifiers at the checkout edge, propagate them on every internal call, and preserve them when work moves to a queue. A new trace ID in the payment worker severs the only cross-service join available in a lightweight tracker.

## Correlation is not distributed tracing

Once events arrive centrally, search by `trace_id`, inspect the error group, and then use the same `trace_id` or `span_id` to jump into logs. Infrai exposes error search and group detail for that investigation. Its useful architectural property is mundane: it is a plain REST API, so Python and Node.js services can send HTTP requests without installing or upgrading a vendor client library. The public discovery surface requires no key and describes request schemas, response schemas, billing, and runnable examples; the live snapshot contains 295 capabilities across 20 modules, with examples in 10 languages.

The second advantage is administrative rather than technical: Infrai uses a single API key and consolidated billing across the capability surface, amounting to one key, one wallet, and one bill. For the checkout owner, that single credential means error events and other backend calls can retain a consistent authentication boundary and cost record instead of rotating many service-specific keys and reconciling many vendor invoices at month-end. Per-call cost, vendor, latency, and request metadata support attribution without making price the design criterion.

Do not mistake those joins for a trace. Infrai has no distributed tracing query and no span tree, so ordering several services' work and identifying the causal parent remains manual. It also has no alert or notification route; threshold checks require polling and a separately operated notifier. Source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, and synthetic or heartbeat monitoring are outside this design. A checkout job that never ran emits no exception, so pair the sink with a dead-man's-switch product such as Healthchecks.

This boundary changes the recommendation. For three or four services with modest traffic and engineers who can follow an identifier through logs, the smaller surface may be exactly right. For dozens of services, fan-out, asynchronous retries, and latency diagnosis, manual correlation becomes operational debt.

Storage is not the hard part there. Causality is.

## Compare operational fit, not feature counts

The fair comparison is between investigation models, not logo lists. Each option can receive application failures, but they ask teams to pay different kinds of complexity.

| Option | Strong fit | Limitation for this checkout design |
|---|---|---|
| Infrai | Small mixed-language services that want one plain REST surface and explicit per-call metadata | Manual `trace_id`/`span_id` correlation; no span tree, built-in notifications, source maps, replay, or heartbeat monitoring |
| Sentry | Error-first investigation where issue grouping, releases, stack-oriented debugging, and tracing belong together | SDK integration and product-specific event behavior create a larger client contract than a neutral HTTP envelope |
| Datadog | Teams already operating a broad logs, metrics, and APM platform that need service maps and connected telemetry | The platform and tagging model require more governance; cost attribution still depends on disciplined service tags |
| Honeycomb | High-cardinality, trace-centered exploration of unfamiliar distributed failures | Teams seeking only a compact exception inbox must still adopt tracing concepts and instrumentation |
| Healthchecks | Scheduled checkout reconciliation or settlement jobs where silence is the failure signal | It complements error tracking; it does not replace exception grouping or cross-service log correlation |

Sentry is the natural comparison when exceptions and releases are the center of gravity. Datadog makes more sense when checkout telemetry must live beside an established infrastructure monitoring estate. Honeycomb is compelling when engineers need to ask new questions of rich traces rather than navigate predetermined dashboards. None of those product choices repairs an unstable schema.

I would choose the lightweight REST sink only under an explicit rule: the team can explain a failed checkout with the envelope, an error-group lookup, and correlated logs within its response target. If that test fails because engineers need parent-child timing, service topology, or automatic trace navigation, use a full APM or tracing product. This is a capability boundary, not a maturity judgment.

Cost attribution deserves the same skepticism. Attribute ingestion and investigation volume by `service`, `environment`, and `release`, preserve the backend's per-call metadata where available, and review unattributed events as schema defects. Do not charge a shared gateway merely because it reported a downstream exception. The owning service is the component that failed or violated the contract, while the trace records where the symptom surfaced.

## Roll out without losing the evidence

Begin with one checkout path and two producers: the public API and the payment worker. Run the envelope beside existing reporting, compare event counts by service and release, and sample correlated logs for redaction mistakes. Reject nonconforming events at the internal boundary during development; in production, count and quarantine them so a schema error does not hide the original checkout failure.

Then add inventory and notification services, one at a time. Keep `event_id` stable across delivery retries, retain the original `occurred_at`, and test a deliberately duplicated event. Also test silence: stop a scheduled reconciliation job and verify that Healthchecks or an equivalent heartbeat monitor detects the absence, because an exception sink cannot capture code that never executed.

The migration trigger should be written down before volume creates attachment to the first tool. Move to trace-native APM when manual correlation repeatedly misses the response target, when service fan-out makes parentage ambiguous, or when latency rather than exceptions becomes the dominant question. Until then, a common schema plus a small capture boundary gives a mixed stack a durable center without pretending that searchable IDs are a tracing system.

## Sources

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Sentry tracing documentation](https://docs.sentry.io/concepts/key-terms/tracing/)
- [Datadog APM documentation](https://docs.datadoghq.com/tracing/)
- [Honeycomb distributed tracing documentation](https://docs.honeycomb.io/get-started/start-building/application/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [ClickHouse documentation](https://clickhouse.com/docs)
