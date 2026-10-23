# 2FA Login SMS OTP API: Evidence Boundaries for Compliance Review

**Short answer:** a 2FA login SMS OTP API should expose an append-only evidence ledger with explicit resend and cancel transitions; an app builder using Node.js should treat delivery as an observation, not proof of receipt.

A compliance notice should be treated as an evidence-producing workflow, not as a successful request to a messaging API. The decision rule is straightforward: choose the design that can reconstruct who authorized the notice, what payload was sent, which delivery transitions were observed, and where the record became unknowable. That rule matters more than a provider's delivery percentage.

For a developer-tools team, the durable boundary is an append-only event record backed by an idempotent dispatcher. The dispatcher may retry a submission, but it must never rewrite the business event that caused it. Store the notice identifier, recipient reference, policy version, payload digest, authorization timestamp, submission attempt, provider response metadata, and later delivery events. Redact the payload when retention rules require it; retain enough metadata to prove the decision without retaining a secret.

## What evidence boundaries matter in 2FA login SMS OTP APIs?

The useful question is not "Did the API return success?" It is: "Can an independent reviewer follow the notice from policy decision to observed outcome?"

There are four invariants. First, the notice has one stable identifier even if transport is retried. Second, each state transition is immutable and ordered by the service's event time plus an ingestion sequence. Third, a duplicate callback does not create a second compliance action. Fourth, an unknown outcome stays unknown; a timeout is not silently promoted to delivered.

A resend is a new attempt linked to the same logical notice, with its own idempotency key and reason. Cancel is a state transition that prevents future work; it cannot erase an SMS already accepted by the transport. For an OTP login, that distinction also prevents an old code from being treated as current merely because a newer code was requested.

This is a deliberately conservative interpretation of evidence. A request acceptance proves that the transport accepted a submission, not that a handset received it. A delivery callback proves what the transport reported, not what a human read. Those distinctions are the difference between an audit trail and a success counter.

Unknown is a state.

That boundary is the point.

Consider the awkward sequence that makes dashboards lie: the worker submits an OTP SMS, the network drops before the response is stored, the worker retries, and the first delivery callback arrives after the second attempt has been accepted. A mutable row can end at `delivered` while hiding the duplicate attempt and the uncertainty that caused it. An evidence ledger keeps the submission-started event, both attempt keys, the unknown response, and each callback as separate observations. A reviewer can then see why the application allowed a retry, whether cancel arrived before the worker claimed the second attempt, and which event supplied the final operational view. This is more data to retain and more states to explain, but it is also a narrower claim: the system reports what it observed, with enough context to challenge the result.

| Design choice | What it establishes | Failure boundary |
| --- | --- | --- |
| Synchronous request log | Submission was attempted | Timeout or lost response leaves outcome uncertain |
| Callback event log | A transport-reported transition was received | Late, duplicated, or reordered callbacks |
| Append-only evidence ledger | Decision, attempts, and observations can be reconstructed | It cannot prove an unobservable external event |
| Mutable status row alone | Current operational view | History and contradictory observations disappear |

The ledger is the source of evidence. A materialized status row is only a query optimization.

## A narrow critical path

The application should commit the compliance decision before asking a transport worker to send anything. The worker claims one pending action, records an attempt, submits an idempotency key, and records the response as a new event. The callback handler performs the same deduplication check as the worker.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Mapping


@dataclass(frozen=True)
class Notice:
    notice_id: str
    recipient_ref: str
    policy_version: str
    payload_digest: str
    authorized_at: str


def digest_payload(payload: str) -> str:
    return sha256(payload.encode("utf-8")).hexdigest()


def record_submission(ledger, notice: Notice, attempt: int, transport):
    key = f"{notice.notice_id}:{attempt}"
    if ledger.has_event(key):
        return ledger.latest_for(key)

    ledger.append({
        "event_key": key,
        "kind": "submission_started",
        "notice_id": notice.notice_id,
        "attempt": attempt,
    })

    result = transport.submit(
        recipient_ref=notice.recipient_ref,
        idempotency_key=key,
    )
    ledger.append({
        "event_key": key,
        "kind": "submission_observed",
        "notice_id": notice.notice_id,
        "attempt": attempt,
        "transport_status": result.status,
        "transport_reference": result.reference,
    })
    return result
```

The example intentionally does not claim delivery from `submit`. In production, `ledger.append` needs a transaction or an equivalent durable write guarantee, and a worker needs a lease with expiry so a crashed process does not hold work forever. The idempotency key is scoped to the notice and attempt; if the transport has a different idempotency contract, adapt the boundary and retain the local key anyway.

## Which failures belong in the record?

A useful ledger names failure modes instead of flattening them into `failed`. A rejected request may be permanent for the payload or temporary for the channel. A timeout after submission creates an ambiguity window: retrying can duplicate the notice, while abandoning it can leave the user uninformed. A callback can arrive before the original response is persisted. A callback can also be repeated, delayed, or arrive after an operator has marked the case for review.

I would represent these as separate observations: `submission_rejected`, `submission_unknown`, `delivery_reported`, `callback_duplicate`, and `manual_review_required`. The exact vocabulary is a team contract, but the rule is stable: preserve the observation and derive the current state. Do not overwrite a disputed event with the newest convenient label.

For retention, store a digest of the rendered notice and a reference to the policy revision rather than assuming that a later reconstruction will produce the same bytes. DKIM offers a useful general lesson here: signing only helps when the signed data and the verification boundary are explicit. The same discipline applies to an audit record; say exactly which fields are evidence and which are operational metadata. See RFC 6376 for that signing model.

## Rejected option: the single status column

A single row with `status = delivered` is attractive because dashboards become easy. It is valid for a low-stakes operational view where historical reconstruction is explicitly out of scope. It is the wrong primary store for a compliance notice: one retry, late callback, or manual correction can erase the sequence that a reviewer needs.

Keep the status row if it makes support queries fast, but rebuild it from the ledger or reconcile it against the ledger. Alert on impossible transitions, such as a delivery observation with no recorded submission, and measure the age of `submission_unknown` records. The latter is a more honest operational signal than a blended delivery rate.

This design is a poor fit for a disposable login prompt where no audit record is required and a short-lived cache is acceptable. It is also a poor fit if the team cannot operate durable storage or protect recipient references; the evidence ledger adds write paths, retention work, and review tooling. Those are real costs, but removing the ledger does not remove the compliance obligation.

The limitation is deliberate: an evidence ledger increases storage and reconciliation work in exchange for historical accountability. A 15-minute retention window may suit a transient challenge, while a compliance notice needs a retention decision that the team can defend; the right choice depends on the obligation, not on a dashboard's convenience.

The team should test the boundary with a small event matrix: crash before submission, crash after submission, duplicate worker claim, duplicate callback, callback before response persistence, and permanent rejection. A test passes only when the final view is explainable and the ledger still contains the original observations. This is also where tool contracts deserve scrutiny: an agent or worker definition should state its inputs and outputs explicitly, as the tool-use guidance from Anthropic demonstrates, because ambiguous output semantics become ambiguous evidence.

The final decision is therefore architectural: use an append-only, privacy-conscious evidence ledger with idempotent transport attempts, and expose a derived status view for operators. It does not promise proof of human receipt. It gives reviewers something narrower and more defensible: a complete account of the authorization, the attempts made, the transport observations received, and the points where the outside system stopped being knowable.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
