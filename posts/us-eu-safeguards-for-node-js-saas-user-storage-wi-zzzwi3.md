# US/EU safeguards for Node.js SaaS user storage with private presigned links

Short answer: keep SaaS user documents in private, region-bound object stores; authorize each request in the application; and issue a narrowly scoped, short-lived download link only after that check. The storage choice matters, but the durable decision is the custody model: who may read an object, where every copy is allowed to exist, and what evidence remains after the link has expired.

Private is necessary. It is not sufficient.

For a document service, an object key is an implementation detail, while tenant ownership, regional placement, deletion obligations, and access evidence are the actual contract. A public-read setting fails that contract immediately. A private container with a long-lived signed URL fails it more quietly, because the URL becomes a bearer credential that can be copied into a browser history, a support ticket, or an observability event. The recipient does not need an application session while the signature remains valid.

## How should SaaS document storage issue private download links in US and EU?

Start with a tenant record that contains a fixed home region and an immutable internal tenant identifier. Resolve that record before selecting a storage client or an object name. Keep the object layout deterministic, such as `tenants/{tenant_id}/documents/{document_id}`, but never infer authorization from a prefix alone: a caller can know a plausible path without owning the document.

The request path should first authenticate the user, then check the tenant and document relationship in the application database, then record the decision, and only then create a download capability for that one key. Bind the response disposition and content type when the interface supports them, so a document intended for download does not acquire a surprising browser rendering path. A short expiry constrains accidental disclosure, although it cannot retract a link already copied during its valid window.

The division is deliberate: the application remains the policy decision point; object storage remains the byte-serving system. It also gives the audit record a meaningful subject and object identifier instead of an opaque successful storage request. NIST's HIPAA Security Rule implementation guidance is a useful reminder that safeguards are administrative and operational as well as technical; a private setting alone does not create an access-control process.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class DownloadGrant:
    tenant_id: str
    document_id: str
    region: str
    expires_in_seconds: int


def make_grant(tenant_id: str, document_id: str, region: str) -> DownloadGrant:
    # The caller has authenticated the user and checked document ownership.
    return DownloadGrant(tenant_id, document_id, region, expires_in_seconds=120)
```

This is intentionally not a storage SDK tutorial. The signing call varies by interface, while the ordering above should not: authorization and audit happen before a capability is minted, and the selected regional client must come from the tenant record rather than a request parameter. Test both sides of that boundary. A useful integration test proves that a user from tenant A cannot obtain a grant for tenant B's identifier, and another proves that an EU tenant selects only the EU storage configuration.

## The hidden failure modes are copies, retries, and logs

Residency is often described as a region selector. In practice it is a copy inventory. The primary object, its replicas, backups, export jobs, malware-scanning inputs, and incident snapshots need the same placement rule. Cross-region replication may improve recovery objectives, but it also creates another regional copy; it is unsuitable where a residency commitment forbids that destination. Record the permitted recovery location next to the tenant's home region, then make backup and restore jobs consume that same record.

A practical review walks the complete path rather than approving a bucket setting: a browser starts an upload, the application associates it with an authenticated tenant, a worker scans or transforms it, an operator exports it for support, a backup process copies it, and a user later requests deletion. At every transition, ask four unglamorous questions: which system can now read the bytes; what identifier joins that copy to the source document; which region receives it; and what event will prove its removal or retention. The answers are frequently split between an application database, deployment configuration, an object lifecycle policy, and an observability pipeline. That split is manageable only if one metadata record is treated as authoritative and every worker consumes it. A new export job should not get to choose a default region. A support workflow should not receive a raw download URL. A restore drill should demonstrate placement and authorization, not merely that a backup archive exists. The expensive mistake is treating these as unrelated feature details until a deletion request or residency review forces someone to reconstruct the path from logs.

Copies accumulate.

Deletion and retention pull in opposite directions. Version history and retention controls reduce the damage from overwrites or accidental removal, yet they can preserve prior data after a product workflow says "deleted." Treat erasure-capable documents and retention-bound records as separate classes, with separate lifecycle policies and explicit legal review. Don't let a prefix decide this by accident.

Multipart uploads create a different kind of invisible copy. The S3 multipart upload model stores uploaded parts until the upload is completed or aborted, so a client that repeatedly disconnects can leave chargeable, inaccessible-looking data behind. A lifecycle policy that aborts incomplete uploads after a period chosen for the product's upload behavior is an operational control, not housekeeping. Monitor the count and age of incomplete uploads, exercise resume behavior on poor connections, and alert on a growing gap between application-level document records and stored bytes.

Logs deserve the same suspicion. Redact query strings or the signature-bearing parameters before URLs leave the request boundary. Preserve a request correlation ID, tenant ID, document ID, decision outcome, and expiry instead. This is less convenient during an incident, but it avoids turning the log system into an alternate document-access channel.

## What should a private SaaS document download path trade for direct links?

There is no universal best delivery path. Direct signed downloads keep document bytes out of the application fleet and are usually the sensible default for ordinary user files. A service proxy can enforce a policy on every read and produce stronger per-access attribution, but it makes application capacity, timeouts, and egress part of the download path. A signed edge distribution can help genuinely shared material; for documents read once by one tenant, its cache economics may not justify the extra control plane.

| Pattern | Authorization moment | Revocation before expiry | Main operational cost |
| --- | --- | --- | --- |
| Application proxy | Each read | Immediate for new reads | Application bandwidth and concurrency |
| Direct signed link | When the link is created | Usually waits for expiry | Careful TTL and URL-redaction discipline |
| Signed edge token | When the token is created | Depends on token and cache design | Cache policy and invalidation operations |

The catch is that a short-lived link is not suitable when policy requires an immediate decision for every byte-serving request or a verifiable record of each completed read. Use a proxy for that narrower class of document, size it for streaming, and set explicit maximum file and concurrency limits. For less sensitive documents, the direct path removes a whole application hop without relaxing the authorization decision that precedes it.

I'm not sure a single retention period is ever defensible across every document category; the answer depends on the legal purpose, the deletion promise, and the recovery objective, and those inputs should be written down before storage configuration is automated.

## Roll out the custody model without a migration weekend

First, add region and document-class fields to the authoritative metadata record, then have new writes use the region-derived destination and private policy. Next, migrate existing objects with a resumable job that records source version, destination version, checksum result, and cutover state for each document. Verify reads through the new authorization path before switching a tenant, keep a bounded rollback window, and remove the old copy only after the retention policy permits it.

Keep the rollout measurable: grants issued, authorization denials, expired-link retries, incomplete upload age, bytes by region, and migrations awaiting verification. Those signals expose the places where the document contract and the storage implementation diverge. The resulting system is less about choosing a storage label than consistently enforcing private access, regional custody, and accountable delivery.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://csrc.nist.gov/pubs/sp/800/66/r2/final
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition
