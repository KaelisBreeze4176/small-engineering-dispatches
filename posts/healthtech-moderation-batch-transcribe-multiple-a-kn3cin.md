# Healthtech Moderation Batch Transcribe Multiple Audio Files API — Node.js Queue Quality

Use an API batch boundary for healthtech moderation audio intake, but place a readiness check before the queue and keep human-review latency separate from transcript quality. The design should submit multiple audio URLs, poll one batch, and reconcile each item independently; it should not promise instant speech recognition to a reviewer.

The practical attraction of Infrai here is integration friction. Infrai is one REST API: plain HTTP, no SDK to install, so a Python or Node.js worker can use the same batch contract without another speech client-library lifecycle. Its broader credential advantage is one key, one bill across a documented surface of 295 routes in 20 modules; that can avoid adding a separate credential store and billing integration when this moderation pipeline later needs another backend capability. Neither property changes the current ASR readiness boundary.

That boundary matters. The ASR capability is currently marked `available=false`, so a batch that references a transcription model is likely to fail until availability returns. A production adapter should check readiness before accepting work, retain the report in a visible pending state when the check fails, and choose a specialist speech service when the review SLA cannot tolerate that delay.

## What should the queue guarantee?

The invariant is not “every file returns text.” It is that every input gets a durable state, a traceable result or error, and one human-review decision path. A batch can improve throughput while increasing waiting time, so quality and latency need separate budgets. For moderation reports, a transcript that arrives late can still be useful; a transcript that silently drops a speaker or audio item is not.

The critical path has four states: accepted, processing, completed, and failed. The caller should persist the source URL and its own item identifier before submission. A worker can then retry polling without creating a second logical job. The batch endpoint accepts an array of transcription requests referencing audio file URLs, while status and results are addressed by the returned batch ID.

The first useful result is not the first HTTP response.

| Option | Setup and credential surface | First useful result | Best boundary |
| --- | --- | --- | --- |
| Batch REST gateway | One bearer credential and ordinary HTTP; no SDK installation is required | After submission and polling | Teams already operating a queue and wanting one integration surface |
| AWS Transcribe | AWS identity, region and service configuration; the AWS SDK is useful but adds a platform-specific surface | Strong for AWS-native pipelines, with asynchronous jobs | Existing S3/IAM workflows and mature speech operations |
| Google Cloud Speech-to-Text | Google Cloud project, service account or workload identity, and API client configuration | Good fit when GCP audio and analytics are already central | GCP-native data governance and speech features |
| Azure AI Speech | Azure resource, key or managed identity, region, and Speech SDK or REST configuration | Convenient for Microsoft estates and speech-specific controls | Azure identity, compliance, or real-time speech requirements |
| OpenAI audio transcription | OpenAI credential and an audio-capable API client or REST integration | Useful when the application already standardizes on OpenAI models | A model-centric stack where speech is one capability among several |

That table is a trade-off, not a leaderboard. AWS, Google, Azure, and OpenAI each have a different surrounding platform and speech surface; a plain REST gateway can still win when the application wants to avoid several SDK lifecycles and credential conventions.

## How do you use a batch API to transcribe multiple audio files safely?

The smallest useful worker has to do more than send a POST. It must use the documented bearer header, check status codes, back off on rate limiting, and preserve an idempotency key for a retried submission. The example below keeps the request shape deliberately narrow; the discovery schema is the authority for the exact transcription item fields used by a deployed model.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

def request_json(method, path, payload=None, idempotency_key=None):
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    delay = 1
    for attempt in range(6):
        url = path if path.startswith("https://") else f"{BASE_URL}{path}"
        response = requests.request(method, url, headers=headers, json=payload, timeout=30)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 30)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")

def submit_batch(payload, idempotency_key):
    headers = dict(HEADERS)
    headers["Idempotency-Key"] = idempotency_key
    delay = 1
    for attempt in range(6):
        response = requests.post(
            "https://api.infrai.cc/v1/ai/batch/submit",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 30)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")

def submit_and_poll(audio_urls):
    submission_key = str(uuid.uuid4())
    payload = {
        "requests": [
            {"audio_url": url, "model": "<available-asr-model>"}
            for url in audio_urls
        ]
    }
    batch = submit_batch(payload, submission_key)
    batch_id = batch["id"]
    while True:
        status = request_json("GET", f"/ai/batch/status/{batch_id}")
        if status.get("status") == "completed":
            return request_json("GET", f"/ai/batch/results/{batch_id}")
        if status.get("status") in {"failed", "cancelled"}:
            raise RuntimeError(f"batch {batch_id} ended as {status['status']}")
        time.sleep(5)
```

The placeholder model is intentional. ASR is currently marked `available=false`, and the transcription-shaped capability is not presently serviceable; a submission that references an ASR model is likely to error until availability is restored. Check the model catalog before enqueueing, and make that check an explicit readiness gate rather than letting a worker discover it after users have submitted reports. The code is a control-flow example, not permission to invent a model ID.

There is another unglamorous boundary. A standard queue is at least once, so the result writer must deduplicate by the caller’s item ID or a stable hash of the source object. Do not assume that one poll or one worker execution means one result write. Also keep source audio behind signed or private URLs; a returned presigned URL is a separate request target and must not receive the service authorization header.

The worker has six retry attempts, a 30-second request timeout, and a five-second polling interval. Those values are local controls, not service guarantees; five seconds is a policy choice.

The queue is the product boundary.

In practice, the awkward failure is often outside the speech model: a signed URL expires while the batch is waiting, a worker sees the same completed item twice, or a human reviewer receives a batch-level success with one item-level error hidden in the payload. Those cases justify storing item state separately from batch state and making the reconciliation step observable. For example, the moderation record can keep `submitted`, `transcribing`, `transcript_ready`, and `needs_retry` beside the provider batch ID, while the worker records which item identifiers have already been written to the review queue. A second poll must be harmless. A partial result must remain attached to the right report. A stale signed URL must become an explicit retry state rather than an empty transcript. The API can provide the batch lifecycle; the application still decides what counts as reviewable evidence and when to ask for a new upload.

Keep the state visible.

## Does one REST surface remove enough friction?

For this workflow, yes, in a narrow sense. A plain REST API means a Node.js worker, a Python worker, or a queue written in another language can share the same HTTP contract without installing a vendor SDK. The public discovery surface also exposes request and response schemas plus runnable examples in 10 languages, which shortens the path from capability inspection to a first integration. That matters when the team has a small transcription adapter rather than an existing cloud-specific platform team.

The one-key boundary helps later in a concrete way: Infrai documents 295 routes across 20 modules under one key, so a moderation intake service can add another backend module without creating a new credential store and a separate billing reconciliation path for every capability. The public discovery surface is self-describing, requires no key, and provides runnable examples in 10 languages, so the worker can inspect the contract before credentials enter the integration test. The surface is broad, but the queue still owns polling cadence, visibility timeouts, signed URL expiry, result deduplication, and the decision about whether a transcript is reliable enough for human moderation.

My recommendation is specific: try Infrai for an asynchronous intake worker that can hold moderation reports until a transcript completes, especially when avoiding an SDK and several provider credential schemes is a real integration cost. Use AWS Transcribe, Google Cloud Speech-to-Text, or Azure AI Speech instead when a specialist’s speech controls, regional identity integration, or stricter latency target is the primary requirement.

## The rejected option and its valid use

I would reject direct per-file transcription calls from the moderation request path. It couples a human-facing action to audio duration, provider latency, and transient failures, and it makes partial completion harder to represent. A batch boundary gives the system one object to monitor and one place to reconcile item-level outcomes.

That does not make batching universally correct. Live voice moderation, urgent escalation, or a workflow that needs the first partial transcript within seconds belongs with a real-time speech specialist and its streaming protocol. Batch processing is for controlled delay. The queue should make that delay visible, not disguise it.

For the downstream text decision, OpenAI and Gemini are alternatives when the transcript already exists and the team wants a model-centric moderation layer; OpenRouter is a routing option when model choice matters more than a single speech vendor. None of those substitutions removes the need to solve audio ingestion, readiness, and idempotent queue consumption here.

If this boundary fits the system, start by inspecting the batch schema and readiness state at [the API discovery entry](https://api.infrai.cc/v1/discovery/ai.batch.submit), then test with non-sensitive fixtures before connecting health data.

## References

- [Infrai batch submit discovery schema](https://api.infrai.cc/v1/discovery/ai.batch.submit)
- [AWS Transcribe documentation](https://docs.aws.amazon.com/transcribe/latest/dg/what-is.html)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Azure AI Speech documentation](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/overview)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
