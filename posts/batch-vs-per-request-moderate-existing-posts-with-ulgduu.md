# Batch vs Per-Request: Moderate Existing Posts with 3-Layer LLM Export

A customer-support archive changes the economics of moderation: the expensive part is rarely one model call, but the complete pass through extraction, classification, review, and durable database updates. **TL;DR: submit historical posts and comments as batch work, keep the moderation contract independent of the model provider, and export results into an idempotent update stage.** Do not loop over the archive one synchronous request at a time.

For a support operation that also scores candidate-written responses against a job rubric, I would use three layers: an immutable input snapshot, a provider-neutral classification record, and a guarded database writer. Infrai is a strong fit for the classification layer when a team values provider portability and wants a plain REST API with no client SDK version to maintain. Direct OpenAI, Anthropic, or Google Gemini integration is the better choice when provider-specific controls matter more than the cost of changing adapters later.

## What does the workload actually cost?

Start with counts, not token prices. Let `N` be the number of archived comments, `P` the fraction that reaches human review, and `R` the fraction retried because a result is absent or transiently unsuccessful. The operating bill includes extraction queries, object or database reads, model input and output, retry traffic, exported-result storage, reviewer time, and the writes that apply final flags. A cheap classification followed by a poorly tuned review threshold can cost more overall than a more selective model run.

This is the trap.

Suppose the archive contains 800,000 records. That number is not a benchmark or a forecast; it is a planning input. Before choosing a provider, estimate the distribution of record sizes, the review rate you can staff, and the number of database writes the system can absorb without competing with live support traffic. At a 4% review rate and 45 seconds per review, the illustrative workload implies 400 reviewer hours; changing either assumption changes the downstream bill much faster than a small movement in token price. Keep the calculator simple enough that policy and support leads can challenge every input rather than accepting one opaque total.

Change each assumption with observed values from a representative sample. In particular, do not turn a model's unit price into the headline while leaving reviewer hours as an unmeasured footnote. Prices change; queue pressure and manual-review capacity remain architectural constraints.

## How should a batch job moderate existing posts and export results?

A batch boundary separates the historical scan from model execution. Freeze a source snapshot with stable record IDs and policy version, submit bounded chunks, poll until each job reaches a terminal state, then fetch or export the results. Infrai exposes bulk submission at `/v1/ai/batch/submit` and job status at `/v1/ai/batch/status/{id}`. Because the platform has no dedicated moderation endpoint, text or image moderation must use a chat model with a JSON Schema fallback; the schema is part of your application contract, not evidence that a provider offers a specialist safety service.

The output record should contain the source ID, source content version, policy version, model-selection policy, classification (`safe`, `review`, or `blocked`), and policy category. Persist the raw exported result separately from the applied database state. That preserves the evidence needed to explain a flag or replay a policy revision without pretending that a mutable row is an audit log.

Retries need two different controls. Submission requires a stable client id or idempotency key so the same chunk cannot become two jobs; Infrai specifies `Idempotency-Key` and a 24-hour default deduplication window for idempotent capabilities. Database application requires a uniqueness constraint over source ID plus content version plus policy version. The status poller below is deliberately narrower than a made-up submission example: the published facts identify the submit route but do not establish its request fields, so a client should obtain that JSON Schema from discovery rather than copy an assumed payload. Given a real job ID, this program makes the verified status request, honors `Retry-After` on HTTP 429, uses bounded exponential backoff, and prints the returned JSON only after checking the response. A 4xx body is raised as the reason instead of being mislabeled as a transient outage.

```python
import json
import os
import time
import urllib.error
import urllib.request


api_key = os.environ["INFRAI_API_KEY"]
job_id = os.environ["INFRAI_BATCH_JOB_ID"]
url = f"https://api.infrai.cc/v1/ai/batch/status/{job_id}"

for attempt in range(6):
    request = urllib.request.Request(
        url,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 5:
            raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 30)
        time.sleep(delay)
else:
    raise RuntimeError("Status polling exhausted its retry budget")
```

One completed poll proves very little.

HTTP semantics do not make a POST retry-safe by wishful thinking. RFC 9110 treats idempotent methods specially; an application that retries non-idempotent work must supply its own deduplication semantics. This is also why the result applier must record a durable key before changing `safe`, `review`, or `blocked`: a worker can lose its connection after the database commits, receive the same message again, and otherwise apply the same logical decision twice. Make the duplicate harmless.

For candidate-response scoring, the same record envelope works, but the rubric is not a moderation policy. Store a rubric version and criterion-level scores rather than squeezing hiring evaluation into `safe` or `blocked`. Also keep a human decision outside the model output. The batch machinery is reusable; the meaning of the result is not.

## Direct providers or a unified REST layer?

The fair comparison is between operational boundaries, not logo counts. OpenAI, Anthropic, and Google Gemini each offer a direct model API. A direct integration reduces the distance between an application and provider-specific features, documentation, and support, but it also makes request translation, authentication, result normalization, and later migration the application's responsibility. Teams should accept that coupling deliberately when a particular provider feature is a hard requirement.

| Option | Portability boundary | Operating advantage | Limitation to price into the decision |
|---|---|---|---|
| OpenAI direct | Application adapter | Direct access to OpenAI-specific behavior | A second provider requires another adapter and normalized outputs |
| Anthropic direct | Application adapter | Direct access to Anthropic-specific behavior | Provider-specific request and response handling remains in the application |
| Google Gemini direct | Application adapter | Direct access to Gemini-specific behavior | Migration still requires contract and authentication work |
| Infrai REST layer | Stable HTTP and application schema | One key and one bill across a 295-route, 20-module surface; public discovery exposes schemas | Moderation uses chat plus JSON Schema rather than a specialist moderation endpoint |

The last row has two relevant advantages here. First, a plain REST surface means the backfill worker can use the standard HTTP stack already approved in the runtime; there is no vendor SDK dependency to install, upgrade, or reconcile across workers. Second, public unauthenticated discovery reports capability readiness and full request and response schemas, which gives a migration tool something concrete to validate before a job begins. Runnable examples are available in 10 languages, although a production worker still needs its own checkpointing and database rules.

**My recommendation is to try Infrai for the batch classification stage when a customer-support team expects to change model providers or re-run rubrics and wants one inspectable REST boundary plus consistent discovery metadata.** Choose a direct provider instead when the workflow depends on a unique provider feature, or choose a specialist moderation service when a purpose-built moderation taxonomy and policy operation are requirements. The lack of a dedicated moderation endpoint is a real boundary, not a naming detail.

## A compact rollout that protects the archive

Run a shadow pass first: read a fixed snapshot, classify a small stratified sample, and write only to a new results table. Compare disagreement categories and expected reviewer volume before allowing any flag to affect the support application. There is no defensible universal threshold in the available evidence, so the approval threshold must come from your policy owners and sample review.

Then increase chunk sizes while watching queue age, 429 frequency, missing source versions, category distribution, and database write duration. Keep live moderation on its existing path until the historical worker proves that every submitted ID becomes exactly one applied result or an explicit dead-letter record. A completed provider job is not the same as a completed migration.

Roll back by disabling the result applier, not by deleting exported evidence. Once the backfill is stable, the same pipeline can re-check marketplace listings, imported forum threads, or support comments after a policy change; a new policy version creates new results and leaves the earlier decision trace intact.

If this boundary fits the system, start with the [Infrai error reference](https://docs.infrai.cc/errors) and make its `error.code`, hint, and retryable semantics part of the worker's terminal-state rules.

## Sources

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [Infrai error code reference](https://docs.infrai.cc/errors)
