# Cost and Compliance Gate for Private Document Storage in Europe-US Web Apps

For a private document storage layer in a Europe-US web app, deletion evidence, private access, and compliance rules change the decision before a storage price ever does.

Short answer: no object-storage provider is the cheapest compliance-friendly choice in the abstract; compare candidates with the same web app workload, then accept only those that pass tests for private reads, deletion, lifecycle behavior, portability, and the required Europe-US controls.

This architecture decision record treats user documents as private objects, keeps authorization in the web application, and makes cost a measured output. The decision is deliberately conditional. A provider name isn't evidence of a control, and an S3-shaped API doesn't settle residency, processor terms, retention, or recovery.

## What should a private document storage web app test across Europe and the US?

Start with invariants, because they survive vendor changes. Every object belongs to one tenant; an unauthenticated caller cannot read it; a download grant is short-lived and scoped to one object; deletion follows the product's declared retention policy; and logs can connect an application decision to the object operation without recording document contents. If legal hold or immutable retention applies, ordinary deletion is no longer the whole requirement, so counsel and the data owner must define which policy wins.

Start there.

The failure boundaries matter more than an attractive happy path. The application database may commit while an upload fails, leaving a document row with no object. An object write may finish while the database transaction rolls back, leaving an orphan. A user can be removed from a tenant after a download grant is issued. A retry can duplicate an upload unless the object key and state transition are idempotent. Lifecycle expiration can also conflict with a database record that still says the file exists. None of these failures is fixed by picking a familiar logo.

Treat the following as acceptance criteria, not brochure questions:

- Can the account, bucket, and object remain private under the exact access path the application will deploy?
- Which region stores primary data, replicas, backups, logs, and support artifacts, and what contract governs each transfer?
- What are the documented durability, consistency, retention, deletion, recovery, and incident-notification commitments?
- Can the team export objects, metadata, checksums, and audit evidence without depending on a proprietary control plane?
- Which charges appear under the real read/write mix, including operations, retrieval, egress, replication, and minimum-retention rules?

Don't accept a checkbox labeled “compliant” as the answer. The required evidence depends on the application's data classes, jurisdictions, contracts, and threat model — and I'm not sure any static comparison can resolve those facts without the current provider terms plus a legal review.

## Decision: compare the workload, not four logos

AWS S3, Cloudflare R2, Wasabi, and Bunny Storage are reasonable candidates to investigate because the reader named them, but this record does not rank them. Current prices and contractual terms can change, while a web application's request distribution is local and measurable. The useful comparison unit is therefore one representative month plus the controls needed to operate it.

| Decision axis | Evidence to collect | Reject when |
|---|---|---|
| Private access | Policy tests for anonymous, cross-tenant, expired, and revoked requests | Any unauthorized read succeeds |
| EU-US governance | Contract, region map, subprocessors, transfer mechanism, deletion terms | A required location or transfer cannot be evidenced |
| Data integrity | Upload checksum, downloaded checksum, retry and overwrite tests | Corruption or ambiguous overwrite behavior isn't detectable |
| Lifecycle and deletion | Policy configuration, object inventory, deletion evidence, restore procedure | Retention and product behavior contradict each other |
| Portability | S3-compatible operations actually used, metadata export, bulk-exit exercise | Exit needs an untested proprietary path |
| Total cost | Stored bytes, request counts, retrieval, egress, replicas, support, labor | A material charge cannot be modeled or bounded |

“S3-compatible” is a starting claim, not a binary architectural property. Build a compatibility matrix from the operations the application uses, then test those operations against every candidate: conditional writes, multipart upload, range reads, metadata, checksums, listing, deletion, and whatever signing flow protects downloads. Your mileage may vary because applications use different slices of the API.

Price comes last. A so-called cheapest service can become the expensive choice when the application performs many small reads, moves data across regions, retains versions, needs a higher support tier, or consumes engineering time around an unsupported operation. Use provider calculators and contracts with captured dates, then replay the same workload assumptions for all four candidates. Don't publish the result as timeless.

## Put authorization and state transitions on the critical path

The browser should not receive a permanent storage credential. A safer critical path is: authenticate the user, authorize tenant and document access against application state, issue or proxy a narrowly scoped download, and set the response disposition intentionally. MDN documents that `Content-Disposition` controls whether content is displayed inline or treated as an attachment, and that its filename parameters affect the suggested download name. That header is presentation behavior, not authorization.

The focused Python sketch below keeps vendor-specific signing behind a small interface and refuses to return a download until application authorization succeeds. The object identifier is opaque; the original filename is metadata, not a storage path. In production, the repository and signer need concrete implementations, request authentication, audit emission, and tests around revocation races.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Document:
    document_id: str
    tenant_id: str
    object_key: str
    download_name: str
    state: str


class DocumentRepository(Protocol):
    def get(self, document_id: str) -> Document | None: ...


class ObjectSigner(Protocol):
    def sign_private_get(
        self,
        *,
        object_key: str,
        expires_in_seconds: int,
        content_disposition: str,
    ) -> str: ...


def attachment_value(filename: str) -> str:
    safe_name = filename.replace("\\", "_").replace('"', "_")
    return f'attachment; filename="{safe_name}"'


def create_download(
    *,
    document_id: str,
    caller_tenant_id: str,
    repository: DocumentRepository,
    signer: ObjectSigner,
) -> str:
    document = repository.get(document_id)
    if document is None or document.state != "available":
        raise LookupError("document is not available")
    if document.tenant_id != caller_tenant_id:
        raise PermissionError("cross-tenant download denied")

    return signer.sign_private_get(
        object_key=document.object_key,
        expires_in_seconds=60,
        content_disposition=attachment_value(document.download_name),
    )
```

Now test the ugly sequence, not merely `200 OK`: upload an object, verify its checksum, commit the document as available, request a download as the owning tenant, attempt the same request from another tenant, expire the grant, delete the document under its policy, and confirm both the application state and storage inventory. Then interrupt the sequence after the object write but before the database commit, retry with the same idempotency key, and verify that reconciliation finds the orphan without exposing it to a user. Repeat the interruption after the database marks deletion as pending but before storage confirms the operation; the row must remain distinguishable from a completed deletion, because collapsing both states creates false evidence. Use explicit client handling for `403` and `404` because those responses can represent authorization, expiration, absence, or deliberate information hiding; don't turn them into a single vague “storage error.”

Make it fail.

Observability should follow the same boundary. Record an internal request ID, tenant ID, document ID, object-operation class, result, byte count, and latency, while excluding credentials, signed query strings, and document bodies. Alert on authorization denials, checksum mismatches, orphan counts, deletion backlog, and unexpected changes in request or egress volume.

This is boring work.

It is also where a private-document design becomes auditable rather than aspirational.

## Lifecycle rules need a separate proof

AWS documents lifecycle management as rules that can transition or expire objects, and it explicitly warns that lifecycle behavior is asynchronous. That is enough to establish a general architectural lesson: a configured expiration rule should not be treated as immediate proof that an object has disappeared. Application state, inventory, and policy evidence need reconciliation.

Model at least four states: pending upload, available, pending deletion, and deleted. A background reconciler can find rows stuck before availability, objects with no live row, and deletion requests that have not yet reached the evidence threshold defined by policy. Keep the threshold explicit. “The rule exists” and “the object is confirmed absent under the required semantics” are different claims.

Versioning, replication, backup, legal hold, and recovery can extend the effective life of bytes beyond the current object view. The exact semantics must come from the selected provider's current documentation and contract; they cannot be inferred from generic API compatibility. Likewise, browser downloads need deliberate media-type and disposition handling, but neither header prevents a user who legitimately downloaded a file from retaining a copy.

## Rejected option and when it is valid

We rejected choosing a provider from advertised storage price alone. It ignores the workload, omits control evidence, and pushes compatibility discovery into production. It is not suitable when documents contain regulated or contractually restricted data, when cross-region transfers matter, or when deletion and audit claims must withstand review. Stick with the existing approved provider when migration risk and evidence-gathering cost exceed the measured benefit of switching.

The price-led option does have a valid use case: public, reproducible, non-sensitive artifacts with an easy source-of-truth rebuild, a tested exit path, and no material residency or deletion obligation. Even there, measure total delivered cost and run restore tests. Cheap bytes do not guarantee cheap operations.

The resulting decision is conditional but actionable: shortlist all candidates that satisfy the control matrix, run the same conformance and failure tests against each, model the observed workload, and record the evidence date. Select on the whole result. Reopen the ADR when data classification, geography, access patterns, contractual terms, or the compatibility matrix changes.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
