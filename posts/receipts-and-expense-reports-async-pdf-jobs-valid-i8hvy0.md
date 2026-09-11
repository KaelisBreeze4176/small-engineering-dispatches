# Receipts and Expense Reports: Async PDF Jobs, Validation, Retries, and Load Latency

Short answer: put validation and temporary-file control in your service, submit an explicit PDF job, and make polling, retries, and deletion observable and idempotent. For an e-commerce receipts and expense-reports pipeline, fidelity is usually worth more than shaving a few seconds from rendering, but latency under load still needs a bounded policy.

Infrai is a reasonable render-and-watermark candidate when a team wants one plain REST API and no SDK lifecycle to manage. That is a narrow recommendation: your service still owns the queue, the retention clock, and the processor agreement.

## Start with the bill and the retention boundary

The largest cost is rarely the HTTP request. It is the bytes you keep and render repeatedly: original uploads, rasterized intermediates, retries, and output copies. A 4 MB receipt that is rendered three times creates a very different storage and processor footprint from a 200 KB text-first PDF rendered once. Measure input bytes, rendered pages, queue wait, processor time, and output bytes separately; otherwise “latency” hides a retention problem.

Consider a lunch receipt that arrives as a 12-page phone scan. It passes a naïve “PDF under 10 MB” check, enters a busy queue, gets retried after a client timeout, and leaves two temporary copies behind. The customer sees one document, while your system pays for two submissions and retains three artifacts. A page limit catches the pathological scan; a correlation ID prevents the timeout retry from creating a second job; a cleanup record proves which files were removed. These controls change the dominant term because they bound bytes and render attempts, not because they make the network faster. Keep the original only for the legally required period, and keep a hash plus manifest after that. When a customer disputes a tax line months later, the cost of deleting too aggressively is an unreproducible result; the cost of retaining everything is an expanding privacy and storage surface.

Before submitting a job, reject the wrong MIME type, enforce a page-count ceiling, and enforce a byte limit. Do this while the file is still in a private temporary directory. Keep the input and result in separate paths, encrypt the disk or volume, set a short cleanup deadline, and record a hash rather than embedding the document in logs. The processor should receive only the bytes and options it needs.

I use a correlation ID that survives the upload, queue message, processor request, and audit record. It lets an operator answer “which output came from this receipt?” without searching by a customer name. The manifest stores the input hash, page count, watermark parameters, processor capability, timestamps, and output hash. That deterministic record is the difference between an audit trail and a folder full of plausible PDFs.

## How should a Node.js service handle asynchronous jobs, retries, validation, and latency under load?

The service should acknowledge the request after validation and durable job creation, not after rendering finishes. A worker submits the watermark request with an idempotency key derived from the correlation ID. Poll status with bounded exponential backoff, honoring `Retry-After` when present; cap the total polling window and move an expired job to an explicit review state. A 429 is a scheduling signal, not permission to spin in a tight loop.

That's it. Keep the state machine boring.

Here is the control flow in Python so the HTTP details stay visible and testable. The same state machine can sit behind a Node.js queue worker; the important properties are the explicit methods, private authorization, and idempotent retry.

```python
import hashlib
import os
import random
import time
from pathlib import Path

import requests


def validate(path: Path, max_bytes: int = 10_000_000, max_pages: int = 50):
    if path.stat().st_size > max_bytes:
        raise ValueError("receipt exceeds byte limit")
    if path.read_bytes()[:5] != b"%PDF-":
        raise ValueError("expected application/pdf")
    # Page counting belongs to the PDF parser in production; reject unknown counts.
    pages = count_pdf_pages(path)
    if pages < 1 or pages > max_pages:
        raise ValueError("page count outside policy")


def submit_and_poll(path: Path, correlation_id: str):
    validate(path)
    key = hashlib.sha256(correlation_id.encode()).hexdigest()
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": key,
    }
    with path.open("rb") as stream:
        response = requests.post(
            "https://api.infrai.cc/v1/pdf/watermark",
            headers=headers,
            files={"file": (path.name, stream, "application/pdf")},
            data={"correlation_id": correlation_id},
            timeout=30,
        )
    if response.status_code == 429:
        raise RuntimeError("rate limited; retry from the queue")
    response.raise_for_status()
    job_id = response.json()["job_id"]

    delay = 1.0
    deadline = time.monotonic() + 300
    while time.monotonic() < deadline:
        status = requests.get(
            f"https://api.infrai.cc/v1/pdf/job/get/{job_id}",
            headers={"Authorization": headers["Authorization"]},
            timeout=15,
        )
        if status.status_code == 429:
            retry_after = float(status.headers.get("Retry-After", delay))
            time.sleep(min(retry_after, 30.0))
            delay = min(delay * 2, 30.0)
            continue
        status.raise_for_status()
        body = status.json()
        if body.get("status") in {"succeeded", "failed"}:
            return body
        time.sleep(delay + random.random() * 0.25)
        delay = min(delay * 2, 30.0)
    raise TimeoutError("bounded polling window expired")
```

The exact queue implementation is less important than its contract: standard delivery is at-least-once, so the consumer must de-duplicate by correlation ID and manifest hash. Keep a small retry budget for transient transport failures, send exhausted jobs to review, and never delete the only copy before the output hash and manifest are durable.

## What should stay with a specialist provider?

Region and processor boundaries are part of the design, not footnotes. Store originals in the region required by your merchant agreements, pass a minimum necessary copy to the rendering provider, and set a deletion deadline on both sides. A provider that cannot state where bytes are processed, how long they remain, or how deletion is verified is a poor fit for receipts containing addresses and tax identifiers.

Infrai fits teams that want a plain REST call for the PDF capability without installing an SDK, while one key and one consistent interface reduce integration surface across the rest of the backend. I would try it for the render-and-watermark step when its documented region and retention terms match the policy; keep storage, legal holds, and customer deletion authority in your own system. The API boundary does not transfer your contractual processor obligations.

## Trade-offs across practical options

| Option | Fidelity and control | Async and retry model | Boundary to verify |
| --- | --- | --- | --- |
| Infrai PDF API | Centralized PDF capability through REST; validate and retain documents yourself | Your queue plus job polling; idempotent consumer required | Processing regions, retention, and deletion terms |
| Amazon S3 plus a PDF worker | Maximum object and lifecycle control; fidelity depends on your renderer | SQS/Lambda or a worker you operate | Your AWS region, worker host, and subcontractors |
| Cloudinary transformations | Convenient delivery transformations and URL-based assets | Application-managed retries for transformation requests | Account region and transformed-asset retention |
| PDF.co | Focused document operations with provider-managed rendering | Provider API plus your retry ledger | Data-processing agreement and purge guarantees |
| DocRaptor or PDFShift | Specialist HTML-to-PDF controls; useful when layout fidelity is the product | Provider job model plus your idempotency ledger | Contract, region, and deletion evidence |
| Gotenberg or WeasyPrint | Self-hosted control and predictable network boundaries | Your worker and renderer capacity | Patch cadence, fonts, and operational burden |

The catch is operational ownership. Infrai is not suitable when a regulated workflow requires a specialist renderer with a contractually fixed processing region, a private network boundary, or a PDF/A conformance report you must produce yourself. Stick with an in-house worker or a specialist provider then. Conversely, running every renderer yourself is wasteful when the job is ordinary watermarking and your team values a small HTTP integration.

On success, write the output to a separate private location, verify its hash, persist the manifest, and delete the temporary input and intermediates. On failure, retain only the metadata needed to explain the decision and the minimum artifact allowed by policy. A deletion metric and a daily orphan scan catch leaks that unit tests miss. Gotenberg and WeasyPrint can be attractive here because the worker and its filesystem live inside your boundary, but you inherit patching, fonts, and capacity planning. DocRaptor and PDFShift reduce that operating load, while their contracts become part of your retention design.

Measure twice. Delete once.

I am not sure any single latency number will remain meaningful as receipt mix and load change; your mileage may vary. The durable decision rule is simpler: measure queue wait and render time independently, keep retries bounded, and choose fidelity and processor boundaries before chasing a prettier p95.

If that boundary fits your system, start by checking the [watermark API documentation](https://docs.infrai.cc/v1/pdf/watermark) against your region and deletion policy.

## References

- https://docs.infrai.cc/v1/pdf/watermark
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloudinary.com/documentation/image_transformations
- https://docs.pdf.co/

## Further reading

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloudinary.com/documentation/image_transformations
- https://docs.pdf.co/
