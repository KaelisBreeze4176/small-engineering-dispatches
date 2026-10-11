# Speech-to-Text API for SaaS Apps Explained — 5 REST Triage Contracts

TL;DR: Choose a speech-to-text API for a fintech SaaS app only after it passes five contracts: an explicit transcript schema, deterministic validation, region-aware privacy controls, replayable ingestion, and field-level evaluation. A simple REST call from Node.js is useful, and a managed or self-hosted Whisper alternative may fit behind it, but neither interface nor model family is the decision boundary. The winning design turns messy customer audio into a valid routing record without letting a plausible transcript silently become the wrong queue, account reference, or urgency.

This architecture decision keeps audio ingestion separate from ticket classification and treats every external response as untrusted input. It also refuses a common shortcut: comparing services by a headline word-error rate or a changing per-minute price. Those numbers can inform procurement, but neither proves that an utterance such as `freeze card ending zero four one six` will produce the fields that the support workflow requires.

## What should a SaaS app require from a speech-to-text REST API?

The first invariant is structural: downstream code receives one versioned record, not a vendor-shaped payload. The record contains a transcript, language status, confidence when the source supplies it, audio provenance, processing region, retention disposition, and validation outcome. Optional source fields stay optional. Inventing confidence where none exists would make the adapter look tidy while corrupting its meaning.

The second invariant is about boundaries. Raw audio lands under a tenant-scoped object key, is processed from an immutable reference, and is deleted or retained according to the declared policy. A database row does not count as proof that an object was deleted; the storage operation and its audit event must have independently observable outcomes. This is where storage architecture matters more than API ergonomics, because retries, partial failures, and delayed deletion are normal distributed-system states.

The third invariant is semantic: routing consumes validated fields, while the original transcript remains evidence. If the parser cannot distinguish a card-freeze request from a disputed transfer, it emits `needs_review`; it does not guess.

Stop there.

Two more invariants close the loop. Every submission has an idempotency key so a timeout can be retried without creating a second ticket, and every schema or classifier change can be replayed against a fixed evaluation set. The five contracts are therefore schema, validation, locality, idempotency, and replayability.

Failure boundaries should be named before implementation. Upload can succeed while transcription times out. Transcription can succeed while JSON decoding fails. JSON can validate while the transcript is semantically wrong. Classification can be correct while the routing write is duplicated. Deletion can be requested while the object remains pending removal. Each boundary needs a state that can be retried or reviewed; collapsing all of them into `failed` throws away the information operators need.

Consider one ordinary call: a customer says, `freeze the card ending zero four one six, and I don't recognize yesterday's transfer`. The transcript may be perfectly readable yet contain two intents, so a schema-valid `card_security` result is still unsafe; automatic routing must abstain. Now add a client timeout after audio upload. The SaaS app retries, the REST API accepts both requests, and two workers race to create tickets unless the application supplied a stable idempotency key. One worker can finish after a deletion request has begun, producing a normalized record that points at an object already moving through its retention lifecycle. None of these cases is represented by word-error rate. They require explicit states, a unique constraint around the stable key, and an audit event that distinguishes deletion requested from deletion confirmed. JSON Schema Draft 2020-12 can express the record's types and required properties, but it cannot prove that the route matches the utterance or that storage deletion occurred. RFC 9110 defines HTTP semantics, but an application still has to decide which operation is safe to retry. Those limits are useful: they identify exactly where the app's contract begins.

Plausibility is not correctness.

## Decision record and option trade-offs

The decision is to place a narrow adapter behind an internal transcription contract, persist immutable input and normalized output separately, and allow automatic routing only when both schema validation and task-specific checks pass. This is a portability mechanism, but portability is not the primary reason. The reason is containment: an upstream response change must fail at one adapter rather than leak through the ticket system.

That boundary also keeps a Node.js request handler, a Python worker, or another application runtime from becoming the permanent owner of a provider-specific response shape. The implementation language can change. The ticket contract should not.

| Option | Structured-output boundary | Privacy control surface | Principal failure mode | Valid use |
|---|---|---|---|---|
| Direct REST call inside ticket handling | Vendor response becomes the application model | Application and provider configuration are coupled | Response drift or timeout contaminates the request path | A disposable prototype with no automatic account action |
| Internal adapter plus asynchronous worker | One normalized, versioned record | Region, retention, and object lifecycle can be enforced around the adapter | Queue delay and more operational state | Production triage where review is acceptable |
| Self-managed transcription runtime | Contract remains internal | Team controls the runtime and storage boundary | Model operations, capacity, and updates become internal duties | Workloads with staff and infrastructure to own that burden |

The middle option is the default for this scenario because support triage is asynchronous and incorrect routing has a larger operational cost than a modest queue delay. That is a trade-off, not a universal ranking. A self-managed runtime can define a tighter infrastructure boundary, while a direct call can be entirely reasonable during a non-production experiment.

US and EU deployment also needs a data-flow decision rather than a region label pasted onto a diagram. Record where audio is stored, where it is processed, which subprocessors can receive it, how long each copy remains, and how a deletion request propagates. The GDPR principles of purpose limitation, data minimization, storage limitation, integrity, and confidentiality are architectural inputs for EU personal data. A service being reachable from an EU endpoint does not, by itself, answer those questions.

## The critical path in Python

The adapter below is deliberately small. It demonstrates the boundary that matters: validate a normalized record before classification, preserve uncertainty, and derive an idempotency key from stable inputs. The transport and storage implementations remain injected interfaces because their behavior belongs in integration tests, not in an example that pretends network calls cannot fail.

```python
from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
from typing import Literal, Mapping, Protocol

Route = Literal["card_security", "payments", "account_access", "needs_review"]


class Transcriber(Protocol):
    def transcribe(self, object_uri: str, region: str) -> Mapping[str, object]: ...


@dataclass(frozen=True)
class TranscriptRecord:
    schema_version: int
    ticket_id: str
    object_uri: str
    processing_region: str
    text: str
    language: str | None
    route: Route
    needs_review: bool
    idempotency_key: str


def normalize(
    transcriber: Transcriber, *, ticket_id: str, object_uri: str, region: str
) -> TranscriptRecord:
    raw = transcriber.transcribe(object_uri, region)
    text = raw.get("text")
    language = raw.get("language")

    if not isinstance(text, str) or not text.strip():
        raise ValueError("transcription returned no usable text")
    if language is not None and not isinstance(language, str):
        raise ValueError("language must be a string or null")

    route = classify_for_review(text)
    stable_input = f"{ticket_id}\0{object_uri}\01".encode("utf-8")
    return TranscriptRecord(
        schema_version=1,
        ticket_id=ticket_id,
        object_uri=object_uri,
        processing_region=region,
        text=text.strip(),
        language=language,
        route=route,
        needs_review=route == "needs_review",
        idempotency_key=sha256(stable_input).hexdigest(),
    )


def classify_for_review(text: str) -> Route:
    normalized = text.casefold()
    matches = {
        "card_security": any(term in normalized for term in ("freeze card", "card stolen")),
        "payments": any(term in normalized for term in ("transfer", "payment dispute")),
        "account_access": any(term in normalized for term in ("locked out", "cannot sign in")),
    }
    selected = [route for route, matched in matches.items() if matched]
    return selected[0] if len(selected) == 1 else "needs_review"
```

This classifier is intentionally conservative and is not a claim that keyword matching is sufficient for production. It shows the fail-closed behavior: zero matches and conflicting matches both require review. A production classifier can replace it without changing the transcript record, but it still needs the same abstention path.

Do not log the full transcript merely because an exception handler makes that convenient. Log the ticket identifier, adapter version, state transition, latency bucket, response size, and a bounded error category. Access to raw audio and transcript text should follow the same authorization and audit model as the support ticket, rather than inheriting broad access from a generic application log store.

## How should structured correctness be graded?

Build the evaluation set from the job, not from generic dictation. It needs consented or appropriately constructed samples that cover account-reference digit strings, accented English, code-switching, background noise, silence, repeated corrections, multiple intents, and requests that must abstain. Keep the expected routing fields beside each audio item, version both together, and prevent test material from entering training or manual tuning without an explicit governance decision.

Grade at three layers. Transport success asks whether an accepted audio object eventually reaches a terminal state. Schema success asks whether the normalized record validates and preserves required provenance. Task success asks whether route and review decisions match the labeled outcome. A fluent transcript can still fail the third layer.

Use counts that reveal harm: false automatic routes, false review decisions, invalid records, duplicate tickets, and unresolved deletion operations. Slice those counts by language status, audio duration band, input channel, region, and adapter version. An aggregate score can conceal a small cohort that is routed badly; the slices make that visible. No invented threshold belongs in a general architecture note. The acceptable false-route limit must come from the support operation's risk owner, and the test set must be large enough to measure that limit with stated uncertainty.

Observability follows the state machine. Measure time in upload, queued, transcribing, validating, classifying, and routing states separately. Alert on stuck state age and invalid-record rate, not merely HTTP errors. Keep payload samples out of routine telemetry; investigate through controlled access to the retained evidence when policy permits it.

## Rejected option and where it still fits

The rejected design performs transcription synchronously inside the customer-facing ticket request and immediately routes from the returned text. It has fewer moving parts and produces fast demos, but a client disconnect, upstream timeout, or malformed response now sits inside ticket creation. Retrying the page can duplicate work, and retaining enough evidence to audit a wrong route becomes an afterthought.

It still has a valid use case: an internal prototype that stores no sensitive audio, takes no account action, and sends every result to a human. The moment the transcript controls a queue with different permissions or response obligations, the asynchronous record and explicit review state earn their operational cost.

Pricing belongs in the decision record as a workload model, not as the headline. Estimate billable audio duration, retries, storage duration, egress paths, review labor, and capacity work for a self-managed Whisper alternative. Then rerun the model with the observed duration distribution. Published unit prices change; the dimensions of the model are more durable.

The final selection criterion is narrow: choose the implementation that can prove the five contracts under representative audio and failure injection. A convenient REST surface matters. Correct, auditable routing matters more.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://json-schema.org/draft/2020-12/json-schema-core
- https://www.w3.org/TR/trace-context/
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://owasp.org/www-project-api-security/
