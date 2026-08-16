# Node.js Presigned URLs for Private Object Storage Large-File Exports

Large-file throughput changes the answer. **Short answer: keep the export private, authorize it in Node.js, and let object storage deliver the bytes through a short-lived presigned download URL.** The application should decide who may read an export; the storage service should move the file without turning the API into a byte proxy.

This is an architecture decision record for an e-commerce report job that produces a ZIP or CSV for an authenticated customer. The invariants are simple: a customer can receive only that customer's export, the object key is immutable for one job attempt, and a slow download must not occupy an application worker. The failure boundaries matter more than the URL syntax.

Keep the bucket private.

## What failure modes affect Node.js private file export downloads?

First, authenticate the customer and load the export record by tenant and job id. Then check that the job is complete, bind the record to its exact object key, and mint a URL for that key. Never sign a key supplied directly by a browser. A presigned URL is a bearer credential: anyone who obtains it can use it until it expires or the object is removed.

The browser should receive a redirect or JSON response containing the temporary URL, while the application retains the authorization decision and an audit event without the query string. The URL lifetime should cover the expected transfer, not the retention period of the export. Cleanup is a separate control.

Three clocks are easy to confuse: job completion, URL expiry, and object lifecycle deletion. They need not have the same value. A retry can create a new immutable key, and a later cleanup task can delete abandoned attempts. Overwriting `latest.zip` is a poor concurrency protocol because two workers can finish in either order and a previously issued link can then name surprising bytes.

Here is the critical path in Python. It uses the standard presign operation exposed by an S3-compatible client; the surrounding authorization and job state are deliberately explicit.

```python
from dataclasses import dataclass
from datetime import timedelta


@dataclass(frozen=True)
class Export:
    tenant_id: str
    job_id: str
    object_key: str
    complete: bool


def create_download_url(customer_id: str, job_id: str, store, exports) -> str:
    export = exports.find_for_customer(customer_id, job_id)
    if export is None or not export.complete:
        raise LookupError("export is not available")

    return store.generate_presigned_url(
        "get_object",
        Params={
            "Bucket": "private-customer-exports",
            "Key": export.object_key,
        },
        ExpiresIn=15 * 60,
    )
```

The method name is intentionally an SDK abstraction rather than a made-up REST route; the storage provider's documented client supplies the concrete signing implementation. In a Node.js service, the same boundary belongs in a route handler or service method, with credentials kept server-side. A download link is the output of authorization, not the authorization mechanism itself.

## Retention and ownership rules for private object storage exports

For a large export, the useful data path is worker to private object, then storage to customer. Proxying the download through Node.js adds a second network hop and makes application capacity proportional to file size. Direct delivery does not remove the need for backpressure: the producer still needs bounded concurrency, the job record still needs a terminal state, and the client still needs a way to retry a fresh link.

The common failure modes are predictable:

- A worker writes a shared filename and a late retry replaces an earlier result.
- The service signs before checking tenant ownership, turning an object key into an authorization oracle.
- A URL is logged in a request query string, tracing system, or support ticket.
- A retry loop repeats every failure, including a denial that cannot be fixed by waiting.
- A cleanup rule deletes an object while a customer is still downloading it.

Use an attempt identifier in the key, for example `exports/tenant-42/job-18427/attempt-3/report.zip`. Store the selected attempt in the database and sign only that stored key. When a job is retried, the old object can become an orphan for lifecycle cleanup; the database remains the index for ownership and retention decisions.

The transfer path also needs observability that does not capture secrets. Record job id, tenant id, object size, generation or attempt id, signing outcome, and download initiation. Do not record the complete URL. Your mileage may vary on a fifteen-minute lifetime: measure the actual distribution of customer download times and add margin, but do not use a year-long URL to hide uncertainty.

There is a particularly dull race worth spelling out. A customer requests job 18427 while attempt 2 is still uploading; the API must return “not available,” not a link to a partially written name. Attempt 2 then times out, attempt 3 completes, and a cleanup worker sees the first object after the database has selected the third. If the key contains only `latest`, the storage service cannot tell those states apart. If each attempt has its own key and the record names the selected attempt, the API can make one answer at every point: no link, a link to attempt 3, or a fresh link to the same completed object. That is a small state machine, but it is the part that keeps a throughput optimization from becoming a data-ownership bug.

## Which delivery boundary belongs in the decision record?

The choice is not “which URL looks easiest.” It is where authorization, bytes, and recovery guarantees live.

| Boundary | Strength | Trade-off or valid limit |
| --- | --- | --- |
| Private object plus presigned GET | Storage carries the large response; the API keeps tenant authorization. | Link possession is access, and revocation is indirect unless the object is deleted or the signing policy changes. |
| Node.js download proxy | The API can enforce a check on every byte request and hide storage URLs. | Application bandwidth, connection slots, and egress become part of the large-file critical path. |
| Public object or CDN URL | Appropriate for intentionally public assets with no customer-specific authorization. | Not suitable for private customer exports because the URL no longer expresses a short authorization window. |
| Managed archive or immutable retention store | Appropriate when legal retention, version history, or recovery copies are invariants. | More operational machinery than a disposable customer download requires; it does not replace the delivery authorization check. |

The catch is that presigned delivery is not a complete retention system. It does not by itself prove that an export is immutable, replicated across regions, or recoverable after accidental deletion. Stick with a proxy when every request must be mediated, when the storage endpoint cannot be exposed to clients, or when the application must transform the response. Choose an archival system when evidence must survive ordinary export cleanup.

## How should a team test Node.js private export links?

Test the contract at the state boundaries, not only the happy-path signature. A customer from tenant A must not obtain tenant B's key; an incomplete job must not produce a link; an expired link must be rejected by storage; and a repeated request for the same completed attempt should mint a fresh URL without changing the object. Test a second attempt too, because that is where shared `latest` keys reveal themselves.

Browser delivery adds a separate standards concern. CORS controls whether browser JavaScript may read a cross-origin response; it does not decide whether a customer is allowed to receive the object. A normal navigation or download and an XMLHttpRequest can therefore have different browser behavior. Define the origin and exposed headers deliberately, and verify the actual response path with the browser you support.

I would also test operational limits: a customer cancels midway, the worker retries after an upload timeout, the cleanup task sees an unreferenced attempt, and the report is larger than the API's intended response size. These tests expose ownership mistakes earlier than a synthetic “URL returned 200” check.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
