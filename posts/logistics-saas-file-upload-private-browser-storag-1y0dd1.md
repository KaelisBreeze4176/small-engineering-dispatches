# Logistics SaaS File Upload: Private Browser Storage with Presigned Links and CORS

Short answer: keep the document private in storage, let the application authorize the tenant-specific object key, and choose a server proxy for small or heavily inspected files; use a browser direct upload with a short-lived signed URL when the file is large enough that relaying its bytes would become the application's bottleneck. The download path should be separate: authorize the shipment user, then issue an expiring signed URL.

That distinction matters in logistics. A proof-of-delivery PDF, customs form, or driver identity document has an owner, a retention rule, and a delivery audience. It is not merely a blob with a convenient filename. The browser is allowed to carry bytes; it is not allowed to decide which tenant's namespace receives them.

## What evidence should prove a browser upload belongs in private document storage?

Start with the control plane, then choose the data plane. The authenticated application creates a document record, derives an object key from the tenant and record identifier, and records the expected content type, size limit, and retention class. The client may propose a filename for display, but the server should normalize it or keep it outside the object key. A key such as `tenant_482/shipment_9017/proof.pdf` is an authorization result, not user input.

For a server proxy, the browser sends the file to the application and the application writes it to a private object store. This is a good default for smaller documents, malware-scanned intake, and teams that want one request boundary to observe. The trade-off is direct and sometimes painful: every byte crosses application infrastructure, so memory limits, request timeouts, worker concurrency, and egress paths need capacity planning.

For direct upload, the application authorizes the key and returns a short-lived signed upload URL. The browser sends bytes to storage, then reports the attempt to the application. The signature limits the operation; it does not prove that the user was entitled to create the document unless the server made that decision before signing. CORS only tells a browser whether a cross-origin response may be exposed to script. It does not replace tenant authorization, and a correctly authorized upload can still be unusable if the storage origin's CORS policy does not allow the exact origin, method, and headers.

The safe completion rule is simple: a client callback is a request to check, not evidence that the object exists. The application should inspect the object or receive an independently verifiable storage event before changing the document record to `ready`.

## The object is not the document

The first failure is a confused deputy: the API signs a key supplied by the browser without checking its tenant prefix. The second is an orphan: the object arrives, but the browser closes before the database record is updated. The third is a false success caused by assuming that a completed HTTP request means the file has the required size, content type, or checksum. The fourth is an access leak caused by turning a private prefix into a public link because delivery was easier that way. These failures are related, but they need different controls; a CORS fix cannot repair a tenant check, and a database row cannot make a public object private after the fact.

Keep it private.

I put those states in the data model. `created` means an upload was authorized, `transferring` means a client may still be sending bytes, `ready` means the server verified the object, and `rejected` means validation or authorization stopped it. The state machine is more useful than a single nullable URL because it makes reconciliation possible. A scheduled worker can find old `transferring` rows, check the object store, and either promote a valid object or mark the attempt abandoned without trusting a browser that may no longer exist.

There is a boring detail here that prevents expensive ambiguity: use a fresh object key per upload attempt, or make replacement explicitly versioned. Reusing the same key while two browser tabs upload different PDFs makes the final object depend on arrival order. If the business rule is “one proof per shipment,” enforce that in the database and make the object key include the immutable document record. Storage consistency cannot supply the missing business constraint. I have no useful basis for promising that a particular provider's overwrite behavior will match your application's race resolution, so make the application rule explicit and test two concurrent uploads against it.

The browser path also needs an honest retry policy. Retry a failed connection when the operation is safe to repeat, but don't blindly replay a completed upload after a timeout. Check the object first, bind retries to an attempt identifier, and cap the retry window. I use 403 as an authorization signal, 409 as a state conflict, and 429 as a signal to back off; the exact mapping belongs in the storage contract you select, but the important point is that these are different decisions.

## A grant is a policy record

This example shows the part that must remain on the server. `storage.sign_upload` represents the selected object-storage adapter; it is intentionally not a vendor SDK. The server validates ownership and constraints before it creates a signed URL, while the browser never supplies the final key.

```python
from dataclasses import dataclass
from datetime import timedelta
from uuid import UUID


@dataclass(frozen=True)
class UploadGrant:
    object_key: str
    upload_url: str
    expires_in: int


def create_upload_grant(
    *,
    tenant_id: str,
    shipment_id: UUID,
    document_id: UUID,
    filename: str,
    content_type: str,
    size_bytes: int,
    storage,
) -> UploadGrant:
    if size_bytes <= 0 or size_bytes > 25 * 1024 * 1024:
        raise ValueError("document size is outside the intake policy")
    if content_type not in {"application/pdf", "image/jpeg", "image/png"}:
        raise ValueError("document type is not accepted")

    # The authenticated request, not filename, supplies these identities.
    object_key = f"{tenant_id}/shipments/{shipment_id}/documents/{document_id}"
    url = storage.sign_upload(
        key=object_key,
        content_type=content_type,
        expires=timedelta(minutes=10),
    )
    return UploadGrant(object_key, url, expires_in=600)
```

The production version still needs authentication, tenant lookup, a database uniqueness constraint, and post-upload verification. It also needs a download function that checks the requesting user's relationship to the shipment before signing a read URL. The URL is a delivery capability with an expiry, not a replacement for the application authorization check.

## Should a SaaS use direct upload or a server proxy for private file storage?

| Boundary | Good fit | Cost or limitation to accept |
| --- | --- | --- |
| Server proxy | Small PDFs, synchronous inspection, one observable request path | Application workers and network paths carry every byte |
| Direct browser upload with a signed URL | Large documents and high concurrent intake | CORS, browser retries, completion reconciliation, and signed-request expiry must be tested |
| Hybrid intake | Small files are inspected inline; large files are scanned asynchronously | Two operational paths and two sets of metrics must stay consistent |

The hybrid design is often the least surprising for a logistics platform. A driver can upload a small delivery image through the proxy while a carrier sends a large customs archive directly to storage. Both flows still use the same document record, tenant-derived key, validation policy, and download authorization. The transport changes; the control plane does not.

The catch is that direct upload is not suitable when the application must transform or inspect the complete byte stream before storage, when the browser cannot be granted the required CORS behavior, or when the organization forbids client-to-storage traffic. Stick with a proxy in those cases. Conversely, a proxy is a poor fit when file volume consumes the application workers that should be handling shipment updates. Your mileage may vary with file sizes and network geography; measure the bytes through the application rather than guessing from request counts.

Do not choose a storage provider from a feature checklist alone. Compare private-by-default behavior, signed read and write semantics, lifecycle controls, integrity validation, multipart cleanup, audit evidence, regional placement, and the operational effort of configuring CORS. AWS documents lifecycle management for objects, and Google Cloud's storage documentation describes its broader object-storage model; those are useful primary references, but their configuration details are not interchangeable. The adapter boundary should make that difference explicit instead of hiding it behind a misleading “S3-compatible” label.

## How do you preserve access control during rollout?

Ship the state machine and private bucket policy before enabling direct browser upload. Test the real production origin, requested headers, expiry window, cancellation, retry after a lost response, duplicate tab, and an unauthorized tenant key. Test downloads separately: an expired signed link should require a fresh application authorization decision, and a link copied from one shipment should not become a general document browser.

Instrument four timestamps: grant created, transfer observed, object verified, and document made ready. Alert on old `transferring` records and on objects with no corresponding document row. Keep searchable metadata such as shipment number, uploader, retention class, and legal hold in the application database; object listing is not a substitute for an auditable query model.

Lifecycle rules are useful for ordinary retention, but they should not be the only place a legal or operational policy lives. Keep the retention class on the document record, make deletion authorization explicit, and record who initiated it. When a shipment is under hold, the cleanup worker should see that state before it asks storage to remove anything.

The final decision rule is compact: proxy the bytes when inspection and simplicity dominate, sign a direct upload when transfer scale dominates, and use the same server-owned identity and verification workflow for both. Private storage is the baseline. Expiring links are the delivery mechanism.

No shortcut.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- https://www.rfc-editor.org/rfc/rfc9110
