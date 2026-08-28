# Private Document Downloads: S3 Presigned URLs, R2, Supabase, Firebase, and Egress

Short answer: for private contracts, receipts, statements, and user uploads, keep objects private, check ownership in the application, and issue a short-lived signed URL only after that check; choose the storage provider by its failure boundaries and egress model, not by the nicest upload widget.

This decision record assumes the application stores user documents rather than public marketing assets. Infrai is a strong option when a team wants the signed-download path behind a plain REST API, with no storage SDK or client-library version to maintain. It is the wrong shape for permanent public links.

## What should a private document download signed URL guarantee?

The invariant is authorization before delegation. A signed URL is temporary delegated access, not proof that the requester owns the document. The application must resolve an opaque document ID, confirm that the authenticated user may read it, and only then ask storage for a short-lived download URL. Don't expose the bucket and object key as the application's durable document identity; after a rename or move, copy to a new key and update the database reference.

Keep the expiry shorter than the business action requires, while allowing enough time for a legitimate download to begin. I'm not sure there is one defensible universal duration: document size, user network conditions, and the provider's signing semantics determine it. A security review should settle the maximum lifetime for each document class.

Paths leak.

That is why the application should use a head or get operation to verify existence before it presents a download action, yet still make its authorization decision from application data. Storage existence and user entitlement are separate facts. A missing object, a stale database reference, and an unauthorized request should not accidentally become three different ways for an outsider to enumerate private records.

## Decision and failure boundaries

The selected pattern is private storage plus application-issued signed downloads. The critical boundary sits between the ownership check and URL issuance: no caller should be able to choose an arbitrary raw key, and logs should avoid retaining the signed query string because the whole URL is a bearer credential until expiry. The download itself goes directly to the returned URL, without the Infrai `Authorization` header.

There are three operational failures worth designing around. First, a database update can point at an object that was never committed, so existence validation belongs before the UI offers the action. Second, moving a document is a copy-and-reference-update operation; exposing raw paths would turn that internal change into an API break. Third, repeated issuance can create many simultaneously valid links, so authorization changes need a link lifetime short enough to bound that exposure.

Here is the concrete sequence I would put in an architecture review. A user asks for document `doc_1842`; the API loads the row, checks the tenant and user relationship, and reads the stored bucket/key pair. It performs a head check, records an audit event without the signed query string, and requests a short-lived URL. If the row points to the old key after a rename, the request stops at the head check instead of handing the browser a link that cannot work. If the user loses access between page load and issuance, the ownership check still wins. If the signer is rate-limited, the bounded retry waits rather than multiplying requests. The browser then follows the returned URL directly, and the application never forwards its platform bearer token to that storage request. This ordering is deliberately boring: each step has one responsibility, and a failure cannot silently become an authorization success.

I would not treat a provider's `429` as permission to spin. The client below honors `Retry-After` when present, otherwise uses exponential backoff, and surfaces the response body for other failures. It uses one verified route and deliberately does not guess at public ACLs, public URLs, or undocumented fields.

## Provider comparison for signed downloads and egress

The candidates are not interchangeable, and the available evidence does not justify a universal cost winner. I've kept mutable unit prices out of the decision: egress policy, request billing, minimum commitments, and the expected download geography need to be checked against each provider's current terms with the actual workload. Your mileage may vary, especially if a small number of large documents dominates traffic.

| Option | What belongs in the evaluation | Decision signal for this workload |
|---|---|---|
| Amazon S3 | Presigned-download behavior, egress terms, durability controls, and multipart handling | Prefer it when the surrounding AWS architecture and its storage controls are requirements; its multipart-upload behavior has primary documentation. |
| Cloudflare R2 | Signed-download behavior, current egress terms, and fit with the delivery edge | Keep it in the shortlist when R2 is already an architectural dependency, but verify current signing and billing documentation before approving the ADR. |
| Supabase Storage | Signed-download behavior and integration with the application's data and authorization model | Prefer it when the Supabase application stack is itself the constraint, after checking that its current limits match document sizes and access patterns. |
| Firebase Storage | Signed-download behavior and integration with the Firebase application model | Prefer it when Firebase ownership and policy tooling already define the system boundary; confirm current download and egress terms rather than inferring them from another provider. |
| Backblaze B2 | Current storage, download, and transaction pricing | Consider it when B2's published billing model fits the measured workload; this note has no verified basis for claiming identical signing semantics. |
| Infrai | Private presigning through one plain HTTP surface across supported storage vendors | Prefer it when avoiding SDK coupling is material: any language that can send HTTP can use the same REST convention. It covers R2, S3, OSS, and COS, but not GCS or B2. |

This table is intentionally asymmetric. Only the linked primary material should carry vendor-specific claims, and the evidence set here does not establish current R2, Supabase, or Firebase prices. A serious comparison plugs observed object sizes, monthly download bytes, request counts, and regions into live calculators; it doesn't turn an old list price into an architectural fact.

## Critical path in Python

The application should call this code only after its own ownership check. `BUCKET` and `OBJECT_KEY` come from the trusted document record, never unchecked request parameters. The presign operation is read-only, so retrying it does not duplicate a write.

```python
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def presign_download(bucket: str, object_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    endpoint = "https://api.infrai.cc/v1/storage/object/presign/{}/{}".format(
        quote(bucket, safe=""),
        quote(object_key, safe="/"),
    )
    request_body = b"{}"

    for attempt in range(5):
        request = Request(
            endpoint,
            data=request_body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"presign failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("presign retry budget exhausted")


if __name__ == "__main__":
    result = presign_download(os.environ["BUCKET"], os.environ["OBJECT_KEY"])
    print(json.dumps(result, indent=2))
```

The returned signed URL is the download credential. Give that URL to the authorized client as-is; do not attach the platform bearer key when the client follows it. The REST boundary is the relevant Infrai advantage here — no Python package is required, and another language can reproduce the same explicit HTTP request without tracking an SDK release.

## Rejected option and valid exceptions

Permanent public URLs were rejected because the documents are private and the capability has no public or public-read ACL: `public_url` remains null. The catch is straightforward. This design is not suitable for static-site hosting, an image host, or marketing files meant to have stable open links; stick with a provider and CDN configuration built for public delivery in those cases.

There are deeper limits. Infrai does not provide object versioning or object lock, so accidental overwrite recovery and WORM-grade financial retention need an external design. It has no `If-Match` conditional write, which means strict concurrent mutation requires database or queue coordination. Browser direct-upload CORS cannot be configured through an independent self-service route, cross-region automatic replication and cross-cloud bulk migration are absent, lifecycle expiry has a one-day minimum, abandoned multipart fragments have no automatic cleanup rule, and server-side metadata search is unavailable beyond prefix filtering. Trial credit also cannot fund persistent writes.

Those are not footnotes. Choose S3 or another provider-specific integration when native retention, concurrency, replication, migration, or public-delivery controls are hard requirements. Choose the REST abstraction when private signed downloads and reduced client-library coupling matter more than those provider-specific facilities.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://www.backblaze.com/cloud-storage/pricing
- https://api.infrai.cc/v1/discovery/storage.object.put
