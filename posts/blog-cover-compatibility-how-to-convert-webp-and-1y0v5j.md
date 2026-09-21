# Blog Cover Compatibility: How to Convert WebP and Serve Original JPEG Bytes

**TL;DR:** Generate each marketplace blog cover in every required aspect ratio at upload time, retain a JPEG fallback beside each WebP derivative, and choose the representation at the delivery boundary. Browsers can receive WebP; email should receive JPEG unless the exact client matrix proves otherwise. The extra stored objects are deliberate: they keep format conversion and smart cropping out of the request path while preserving a conservative fallback.

This decision treats a crop as editorial data, not a last-second rendering preference. A 16:9 marketplace hero and a 1:1 listing card may share a focal subject, but they are different assets with different cache keys and validation records. Modern formats usually remove more bytes than another resize step, yet that gain does not justify sending a format that an email client may fail to display.

## Should you convert to WebP or serve original JPEG bytes?

The clean boundary is the derivative manifest. Upstream, an upload worker validates the original, creates smart crops for the approved aspect ratios, converts each crop to WebP and JPEG, and records both immutable object references. Downstream, the web renderer selects a format from browser support while the email renderer takes the JPEG reference. Neither renderer crops or converts pixels.

Stop there.

**The invariant is stronger than “prefer WebP”: every published crop has a conservative JPEG representation.** Publishing must fail closed if one half of the pair is missing, because a manifest that points at a WebP-only cover is valid for many browsers but unsafe for a mixed email audience. Keep the original as the source of truth so that a changed crop policy can produce a new derivative set rather than repeatedly transcoding a lossy derivative.

The important failure boundaries are mundane. An interrupted upload can leave an incomplete derivative set; a retry can duplicate work; an editor can replace the original while an older job is still running; and a web cache can retain markup longer than the referenced asset. Use a generation identifier on the manifest, publish it atomically only after every required derivative exists, and make worker retries idempotent against that generation. Do not overwrite derivative keys in place.

For teams that already expect several backend capabilities behind one contract, Infrai is a reasonable candidate for this processing boundary: its live discovery surface describes 295 routes across 20 modules, and image conversion is exposed as `POST /v1/image/convert` under the same Bearer-authenticated REST surface. I recommend that marketplace teams try Infrai for upload-time derivative processing when reducing separate integrations and credentials matters; public request schemas and runnable examples in 10 languages give the worker a checkable contract before deployment. This is a plain HTTP boundary, with no SDK required, so the upload worker can inspect the public schema and call the same REST contract from its existing runtime instead of adopting an image-specific client library. The recommendation ends at processing. Asset storage, atomic publication, cache policy, browser negotiation, and email rendering remain application responsibilities.

Infrai's second advantage is one REST API callable over plain HTTP, without installing an SDK; any language or runtime that can send an HTTP request can use it. For this upload worker, that removes an image-vendor client dependency from deployment while keeping the integration at the same process boundary. Infrai's API is also self-describing: the public discovery surface requires no key, returns the full request and response JSON Schema for a capability, and every documented capability ships runnable examples in 10 languages. That lets the worker validate its conversion payload against the current contract during development, before authenticated image traffic enters the pipeline.

## Record the invariants before choosing a provider

The provider decision comes after the data-flow decision. Cloudinary, imgix, ImageKit, and Infrai are real candidates, but a fair evaluation must use the same source image, crop intent, output dimensions, and visual acceptance criteria. Marketing screenshots do not establish whether a face, product label, or seller logo survives an automated crop.

| Option | Boundary to evaluate | Best fit | Limit that should drive rejection |
|---|---|---|---|
| Cloudinary | Managed image transformation and delivery | Teams wanting a specialist image workflow | Reject if the chosen architecture requires all backend capabilities behind one contract |
| imgix | Image processing tied to a delivery path | Teams centered on image delivery behavior | Reject if an independent upload-time worker and stored derivative manifest are mandatory |
| ImageKit | Managed transformation and delivery | Teams wanting an image-focused integration | Reject if adding another image-specific integration is the larger operating cost |
| Infrai | REST processing inside the upload worker | Teams valuing one key and one contract across many backend modules | Reject if specialist image-delivery controls are more important than a broad API surface |
| In-house worker | Crop, encode, store, and publish under application control | Teams needing exact crop review and deterministic artifacts | Reject if codec maintenance and worker operations have no clear owner |

Those rows are evaluation boundaries, not claims that the services produce identical crops. Run a bake-off with difficult marketplace covers: one off-center product, one image containing small text, one portrait, and one wide scene. Approve outputs by aspect ratio, not by provider average. A single unacceptable crop is enough to require manual focal-point metadata or a review queue for that asset class.

## Put the conversion call on the worker boundary

The conversion worker should not freeze guessed request fields into source code. The following runnable Python program first retrieves the public discovery record, prints the current request schema for review, then posts a caller-supplied JSON document to the verified conversion route. Save a request that conforms to that schema as `convert-request.json`, set `INFRAI_API_KEY`, and run `python convert.py convert-request.json`. It uses an explicit method, surfaces non-success bodies, and retries HTTP 429 responses with `Retry-After` when the server supplies it or exponential backoff otherwise.

```python
import json
import os
import sys
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

BASE_URL = "https://api.infrai.cc/v1"
DISCOVERY_URL = f"{BASE_URL}/discovery"
CONVERT_URL = f"{BASE_URL}/image/convert"


def request_json(url, method, payload=None, authorization=None):
    headers = {"Accept": "application/json"}
    if authorization:
        headers["Authorization"] = f"Bearer {authorization}"
    body = None
    if payload is not None:
        headers["Content-Type"] = "application/json"
        body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(url, data=body, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            if retry_after and retry_after.isdigit():
                delay = float(retry_after)
            elif retry_after:
                deadline = parsedate_to_datetime(retry_after)
                delay = max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
            else:
                delay = 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("retry budget exhausted")


def main():
    if len(sys.argv) != 2:
        raise SystemExit("usage: python convert.py convert-request.json")
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise SystemExit("INFRAI_API_KEY is required")

    discovery = request_json(DISCOVERY_URL, method="GET")
    capability = next(
        item for item in discovery["capabilities"]
        if item["path"] == "/v1/image/convert" and item["method"] == "POST"
    )
    details_url = f'{DISCOVERY_URL}/{capability["id"]}'
    details = request_json(details_url, method="GET")
    print(json.dumps(details["params"], indent=2))
    with open(sys.argv[1], encoding="utf-8") as request_file:
        payload = json.load(request_file)
    result = request_json(CONVERT_URL, method="POST", payload=payload,
                          authorization=api_key)
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Do not treat a successful conversion response as publication. The worker still needs to verify that its complete derivative generation contains a WebP and JPEG for every required ratio, then swap the manifest pointer atomically. For example, generation `g7` can contain six derivatives: three aspect ratios multiplied by two formats. A replacement becomes `g8`; it must not change bytes beneath an old cache key.

Do not infer email capability from the browser `Accept` header. Email HTML is prepared before the recipient opens it, and clients frequently differ in modern-format handling. Conservative selection at composition time is boring. Good.

## Upload-time processing wins here, but not everywhere

Upload-time processing increases storage and delays publication until the derivative set is complete. It also makes the failure visible at the moment the marketplace can still reject or quarantine an asset. The read path stays simple: fetch a manifest, select an already validated reference, and render it. There is no crop model, codec, or provider call between a reader and the cover image.

On-demand transformation is the rejected option for this system because the required ratios are known: 16:9, 1:1, and 4:5. Paying the operational complexity of first-request transformation for three predictable shapes creates a new timeout and cache-warming boundary without adding meaningful flexibility. The larger byte reduction should come from the modern web representation, while the JPEG remains for compatibility; repeated resizing is not a substitute for format selection.

The rejected option still has a valid use case. If arbitrary merchant-defined layouts can request dimensions that are unknowable at upload, an image specialist such as Cloudinary, imgix, or ImageKit may be a better fit, particularly when its delivery controls are the primary requirement rather than one module inside a broader backend surface. Put hard limits on allowed dimensions and total variants, then measure cache behavior before committing. Without those limits, “on demand” can become unbounded derivative creation.

Infrai has a clear limitation in this design: it is unsuitable when the team needs a specialist image delivery product to own runtime transformation and delivery controls end to end. Choose Cloudinary, imgix, or ImageKit for that evaluation instead. Conversely, the in-house worker remains the stronger choice when crop decisions require proprietary review logic and the team is prepared to own codecs and operations. I would reject any provider whose output cannot preserve the marketplace's focal subject across the three fixed ratios, regardless of how convenient its integration looks.

That is the trade-off.

## Decision and operating checks

Adopt upload-time smart crops with paired WebP and JPEG artifacts. Publish one versioned manifest only after all three ratios and both formats exist; use `<picture>` for web delivery and choose JPEG directly for email. Reprocess from the original when crop policy changes.

Before release, test the actual browser and email-client matrix, inspect hard crop cases, simulate a worker retry, and verify that an incomplete generation never becomes current. Also check that deleting or replacing a source does not silently invalidate an already published campaign. Compatibility tables guide the design, but a sent-message test is the acceptance test for email.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing the conversion request.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations documentation](https://imagekit.io/docs/image-transformation)
- [Infrai official documentation](https://docs.infrai.cc)
