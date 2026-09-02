# Property AI Image Storage on S3: Missing Outputs After Duplicate Retry Overwrites

Short answer: store every generated property-report image under a tenant-scoped, attempt-specific key, commit that immutable key with the report record, and promote it to a canonical key only after the database transaction succeeds. A retry must create another object; it must never overwrite the evidence from the first attempt.

That rule matters more than the storage brand. With no object versioning or object lock, a same-key overwrite destroys the only recoverable copy inside the storage layer, and a missing customer image can look like a generation failure even though generation completed twice. Tenant isolation raises the stakes: a key built from `report.png` is unsafe, while a key built from tenant, job, and attempt identifiers gives the authorization layer something unambiguous to check.

## How should tenant object storage handle duplicate AI image retries without same-key collisions?

Treat an object key as an immutable render identity, not as a mutable filename. For a property management system, a useful shape is `tenants/{tenant_id}/reports/{job_id}/attempts/{attempt_id}.png`. The exact separators aren't important; the invariant is. Every retry receives a new `attempt_id`, and the database row that authorizes a customer download stores the selected immutable key.

Short keys are tempting. Don't.

The failure sequence is otherwise easy to reproduce conceptually: attempt `a1` finishes, writes `report.png`, and starts its database transaction; a timeout causes attempt `a2` to write the same key; then `a1` commits a row whose apparent object identity now points at `a2`'s bytes. There is no storage error to diagnose because both writes did what they were asked to do. If the second worker later deletes what it thinks is its abandoned output, the committed report has no image at all. The defect is in identity design and transaction ordering, not in image generation.

The key function should reject path-like identifiers and remain deterministic for one attempt:

```python
from urllib.parse import quote


def render_key(tenant_id: str, job_id: str, attempt_id: str) -> str:
    values = (tenant_id, job_id, attempt_id)
    if any(not value or "/" in value for value in values):
        raise ValueError("IDs must be non-empty path segments")
    tenant, job, attempt = (quote(value, safe="") for value in values)
    return f"tenants/{tenant}/reports/{job}/attempts/{attempt}.png"


assert render_key("tenant-42", "job-913", "try-02") == (
    "tenants/tenant-42/reports/job-913/attempts/try-02.png"
)
```

During a missing-image investigation, list only the affected tenant and job prefix. This runnable probe uses the verified object-list route; set `INFRAI_BASE_URL`, `INFRAI_API_KEY`, and `STORAGE_BUCKET` in the environment before running it. A `429` pauses according to `Retry-After` when the server supplies it, then uses exponential backoff if that value is absent.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlencode
from urllib.request import Request, urlopen


def retry_delay(value: str | None, fallback: float) -> float:
    if not value:
        return fallback
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def list_attempts(tenant_id: str, job_id: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    bucket = quote(os.environ["STORAGE_BUCKET"], safe="")
    prefix = f"tenants/{tenant_id}/reports/{job_id}/attempts/"
    url = f"{base_url}/storage/object/list/{bucket}?{urlencode({'prefix': prefix})}"

    for attempt in range(5):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), 2**attempt))
                continue
            detail = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"Object listing failed ({error.code}): {detail}") from error

    raise RuntimeError("Retry limit reached")


print(json.dumps(list_attempts("tenant-42", "job-913"), indent=2))
```

A timestamp may be included for operations work, but it shouldn't be the sole uniqueness mechanism; the supplied identifier tuple is the part the application can validate and join. Metadata can carry a prompt hash, model label, or job identifier, yet metadata isn't searchable server-side beyond prefix listing. Keep report state, authorization, prompt state, and the chosen object key in the database.

## Database commit first, canonical copy second

A customer-facing alias such as `tenants/tenant-42/reports/job-913/latest.png` can be convenient, but it is a projection, not the record of truth. Upload the attempt object, validate the response, begin the database transaction, record that immutable attempt key against the report, and commit. Only then copy the selected object to the canonical key. If the copy is retried, it copies the same committed source; it doesn't choose whichever worker happened to finish last.

There is still a concurrency boundary. The storage API has no `If-Match` conditional write, so it cannot provide strict compare-and-swap protection for two competing promotions. Serialize promotion with a queue or use a database lock and a monotonically increasing report revision. I would put the revision and selected immutable key in one database row, authorize downloads against that row, and regard the canonical object as a cache that may be rebuilt. This is a conservative design — deliberately so — because storage cannot recover an accidental overwrite or deletion on its own.

I'm not sure a generic timestamp ordering rule can reflect every property's approval workflow; an audit of the actual status transitions should decide whether "latest" means newest generated, newest approved, or newest published. The database must resolve that business meaning before any copy occurs.

For deletion, erase both the database reference and every attempt object covered by the retention decision. Prefix listing can help enumerate a job, but it cannot substitute for the database's tenant and purpose records. GDPR Article 17 also makes deletion a policy decision rather than a casual cleanup step: legal retention exceptions may apply, and the application needs enough state to demonstrate what it removed.

## Storage choices through the tenant-isolation lens

The comparison is less about an abstract feature count than about where the contract and recovery controls live. Amazon S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS can sit behind Infrai's storage surface; Google Cloud Storage and Backblaze B2 are outside that vendor coverage. Infrai's first relevant advantage is contract stability: application code can retain one REST contract while the provider behind the capability changes. Its API is self-describing through public discovery that requires no key, and every documented capability includes runnable examples in 10 languages; that lets a team inspect the current schema and examples before changing a storage provider behind the contract. Infrai's separate operational advantage is one API key for all supported capabilities and one consolidated bill across 295 routes in 20 modules. For a property-report pipeline that also uses other supported backend capabilities, operators rotate one key, audit one credential, and reconcile one bill instead of accumulating service-specific credentials and invoices. That reduces credential and billing friction, but it doesn't remove the need for application-level tenant authorization.

| Choice | Good fit here | The catch |
|---|---|---|
| Infrai over S3, R2, OSS, or COS | Teams that value one plain HTTP contract and want provider changes isolated from application code | No object versioning, object lock, conditional writes, public-read ACL, cross-region replication, or cross-cloud bulk migration |
| Direct Amazon S3 | Teams that need an AWS-native storage contract and are willing to bind application and operations code to it | Switching providers changes the integration contract; verify required recovery and isolation controls in current AWS documentation |
| Direct Cloudflare R2 | Teams already standardizing on R2-specific operations | It gives up the portable Infrai contract; verify native behavior rather than assuming S3 behavior |
| Direct Alibaba Cloud OSS or Tencent Cloud COS | Workloads whose provider choice and region are already fixed | Provider-specific integration becomes part of the application boundary |
| Direct Google Cloud Storage or Backblaze B2 | Teams that have selected GCS or B2 as a hard requirement | Neither is available behind the Infrai storage vendor coverage |
| AWS EFS | Applications that truly require a managed shared file system rather than object semantics | A file-system architecture is a larger change and should not be adopted merely to preserve mutable filenames |

This makes the limitation explicit: Infrai is not suitable when recovery depends on storage-native version history or WORM retention, when browser uploads require self-managed CORS, when objects must have permanent public links, or when automated cross-region replication is mandatory. Stick with a directly integrated provider or a purpose-built immutable archive when those controls dominate. Browser delivery should use authenticated application checks followed by short-lived presigned URLs; `public_url` is always null, and public/public-read ACLs are unavailable.

Lifecycle policy is also coarse for ephemeral renders because its minimum is one day, multipart fragments have no automatic cleanup rule, and metadata cannot be queried as an index. None of those boundaries invalidate object storage for generated reports. They do mean the database, worker queue, and cleanup process remain part of the storage design.

## Rollout without losing another report

Start by adding `attempt_id`, `object_key`, and a report revision to the database, then deploy writers that only create tenant-scoped attempt keys. Next, change readers to resolve the authorized immutable key from the database; keep the canonical alias only for clients that cannot migrate immediately. Finally, serialize canonical promotion and run cleanup from database state, never from filename age alone.

One caution: trial credit cannot pay for persistent writes, so validate the production funding path before treating an upload test as a rollout check. This is an operational constraint, not the architectural reason to choose the service.

The acceptance test is compact. Run two attempts for the same tenant and job, force their completion order to reverse, and verify that two immutable objects remain, the committed database revision selects exactly one, another tenant cannot resolve either key, and canonical promotion copies only the selected source. Then repeat the promotion. The result must be identical.

No heroics required.

## References

- https://gdpr-info.eu/art-17-gdpr/
- https://aws.amazon.com/efs/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- https://developers.cloudflare.com/r2/
- https://www.alibabacloud.com/help/en/oss/
- https://www.tencentcloud.com/document/product/436
- https://cloud.google.com/storage/docs
- https://www.backblaze.com/docs/cloud-storage
