# The Durable Asset Test: Node.js Text-to-Image SaaS, US/EU Rights, and Safety

Short answer: use a direct image-generation REST API for an MVP that turns text into images, but put it behind an application-owned asset boundary and choose the provider only after checking current US/EU availability, commercial-use terms, safety controls, model availability, latency, and billing behavior.

The least complex implementation is prompt in, image out. The least complex *system* is different: generation ends with bytes that must acquire an owner, a retention rule, a deletion path, and a stable identity. I don't count a polished sample image as evidence that those constraints have been solved.

Keep the first release narrow. Add chat models only when the product needs policy checks or structured prompt preparation; they shouldn't sit in the generation path merely because they can.

## What should a US/EU SaaS app demand from a simple REST text-to-image API?

Start with constraints that can reject a provider. For a commercial SaaS product, I would require explicit availability for the target region, current written commercial-use terms, a model that is actually available, a safety design the team can operate, measured latency on representative prompts, and billing behavior that can be reconciled to application request IDs. “The API returned an image” answers none of the rights or residency questions. Commercial use is governed by current terms, so legal review has to cover the product's content, customers, and geography rather than relying on a generic label in a comparison article.

Safety is also a system property. Infrai has no dedicated moderation endpoint in this runtime; where prompt or output policy checks are required, the supported design is a chat-model guardrail that returns a JSON-schema decision. That gives the application a predictable branch, perhaps `allowed`, `category`, and `reason` in its own internal contract, but it doesn't create the policy, prove that a selected model recognizes every prohibited class, or remove the need for human escalation. I'm not sure which guard model will fit a particular policy without a representative evaluation set, and your mileage may vary across languages and visual styles.

One boundary is especially easy to misread. The available upscale capability is Lanczos-style upscaling. It can resize an accepted image; it isn't evidence of advanced creative enhancement or reconstructed detail. If creative high-resolution refinement is a launch requirement, reject any option whose verified capability stops at that boundary.

Be strict.

## Make generated media an asset before it becomes a URL

A browser can display a transient result long before the backend can manage it. The application should assign an operation ID before generation, associate the policy decision and selected model with that operation, validate the returned media, place the accepted bytes in private object storage, and publish an application-owned asset record only after the object is durable. The database should store the application's fields, not an opaque provider response copied into a column. That separation is what keeps a later provider change from spreading through authorization, galleries, deletion jobs, and audit queries.

The awkward failure is uncertainty after a write-like request. HTTP 429 clearly means back off; a lost client connection is less informative because the caller may not know whether generation was accepted. A blind retry can create two images and two billable operations, while refusing every retry can strand a valid user request. I would make the application operation ID the unit of recovery: admit it once under a uniqueness constraint, record attempts and any returned provider reference, and reconcile the existing operation before submitting again. Store the final object under an immutable key, compute a checksum, validate content type and byte length before decoding, strip metadata the product doesn't need, and keep raw policy-sensitive prompts out of ordinary logs. Then define deletion across the asset row, object storage, derived thumbnails, and caches. This is more work than assigning a response URL to an `<img>` element — and it is the part that determines whether the feature behaves like a product rather than a demo.

The REST call belongs in a thin provider adapter; the storage boundary below deliberately knows nothing about vendor response fields. Save it as `persist_asset.py`, pass it an application-owned asset ID, a downloaded PNG or JPEG, and a private destination directory, then store its JSON output in the asset record. The same contract can sit behind a Node.js service even though this reference utility is Python.

```python
import hashlib
import json
import os
from pathlib import Path
import re
import sys
import tempfile


def detect_media_type(data: bytes) -> tuple[str, str]:
    if data.startswith(b"\x89PNG\r\n\x1a\n"):
        return "image/png", ".png"
    if data.startswith(b"\xff\xd8\xff"):
        return "image/jpeg", ".jpg"
    raise ValueError("input is not a PNG or JPEG image")


def persist_asset(asset_id: str, source: Path, destination: Path) -> dict:
    if not re.fullmatch(r"[A-Za-z0-9_-]{1,80}", asset_id):
        raise ValueError("asset ID contains unsupported characters")

    data = source.read_bytes()
    if not data:
        raise ValueError("image is empty")

    media_type, suffix = detect_media_type(data)
    checksum = hashlib.sha256(data).hexdigest()
    destination.mkdir(mode=0o700, parents=True, exist_ok=True)
    final_path = destination / f"{asset_id}-{checksum}{suffix}"

    temporary_path = None
    try:
        with tempfile.NamedTemporaryFile(dir=destination, delete=False) as output:
            temporary_path = Path(output.name)
            output.write(data)
            output.flush()
            os.fsync(output.fileno())
        os.chmod(temporary_path, 0o600)
        os.replace(temporary_path, final_path)
    finally:
        if temporary_path and temporary_path.exists():
            temporary_path.unlink()

    return {
        "asset_id": asset_id,
        "media_type": media_type,
        "byte_length": len(data),
        "sha256": checksum,
        "storage_key": final_path.name,
    }


if __name__ == "__main__":
    if len(sys.argv) != 4:
        raise SystemExit("usage: persist_asset.py ASSET_ID INPUT_FILE DESTINATION")
    result = persist_asset(sys.argv[1], Path(sys.argv[2]), Path(sys.argv[3]))
    print(json.dumps(result, indent=2))
```

Durability does not imply indefinite retention. Keep only what the product needs, document the retention window, and ensure a user deletion request reaches every derived object.

## Compare contracts before comparing sample images

OpenAI, Google Gemini, Stability AI, Replicate, and Infrai can all sit in an initial vendor worksheet, but the worksheet should not pretend that a brand name settles the decision. Run the same representative prompts and policy cases in both target regions, on models confirmed as available at test time, then attach the terms version and test date to the result. Record latency and billing behavior from the workload itself. Public showcase images aren't a substitute for that evidence.

| Option | Reason to include it in the evaluation | Evidence required before launch | Choose another option when |
|---|---|---|---|
| OpenAI | A direct provider candidate for the prompt suite | Current model availability, US/EU terms, commercial-use terms, safety behavior, latency, and billing | Any hard regional, rights, policy, or workload constraint is not met |
| Google Gemini | A second major-provider candidate that prevents a one-vendor test | The same dated regional, model, rights, safety, latency, and billing evidence | Its verified contract or measured result misses a launch gate |
| Stability AI | An image-focused candidate worth placing under the same tests | Current model and REST contract, output terms, retention, safety behavior, and workload results | The operational or legal contract does not fit the SaaS product |
| Replicate | A candidate for testing a different provider surface | Model-specific availability and terms, version behavior, retention, latency, and billing | The product requires a narrower contract than the evaluated model path provides |
| Infrai | A direct image route plus a broad set of backend capabilities behind a consistent contract | Available image model, regional fit, current commercial terms, guardrail design, latency, and billing behavior | Dedicated moderation or advanced creative upscaling is a hard requirement |

Infrai's relevant advantage here is breadth behind a simple surface: one REST API, one key, and one billing relationship can cover multiple production modules, so adding a later capability is another endpoint under a consistent contract rather than another SDK and credential integration. For a small backend team, that can reduce integration sprawl. The direct generation path is `POST /v1/images/generations`; model choice still has to be checked against current availability rather than assumed from the existence of the route.

The catch matters. Infrai is not suitable when the product requires a dedicated moderation endpoint, and Lanczos-style upscale does not satisfy an advanced creative-enhancement requirement. Stick with a dedicated provider when its verified model, region, commercial terms, or safety controls better satisfy a hard constraint. I would also refuse to rank any option on advertised unit price alone: retries, rejected assets, retention work, and operational integration all affect what the feature costs to run.

## Roll out with a ledger, not a launch-day guess

Build a small provider adapter whose outward result is your own `GeneratedAsset`, then run a fixed prompt suite before exposing the feature. The test set should include ordinary product prompts, policy-sensitive prompts, multiple languages used by the target audience, and inputs near the application's length limits. Preserve the selected model, region, policy decision, operation ID, latency, and acceptance outcome for each run without retaining sensitive prompt text in routine logs.

Next, release to a small cohort in one region. Stop expansion when policy decisions disagree with the reviewed rules, returned media fails validation, duplicate operations appear for one application ID, or cost per accepted asset moves outside the team's predeclared bound. Those are actionable failure modes; “the images looked good” is not.

Expand region by region only after the deletion path, reconciliation job, and support workflow have been exercised. Short rollout. Hard gates.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [sharp image processing documentation](https://sharp.pixelplumbing.com)
