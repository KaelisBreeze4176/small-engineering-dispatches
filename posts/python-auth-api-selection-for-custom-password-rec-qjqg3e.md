# Python Auth API Selection for Custom Password Recovery Email (Event Polling)

Short answer: for a gaming account-recovery flow with no webhooks, keep the reset template under application-team ownership, send through a narrow provider adapter, and poll delivery events into a durable suppression ledger before any later message is accepted for that recipient. Supabase Auth, Clerk, and NextAuth may all appear on the initial shortlist, but the decisive question is not the logo on the authentication layer; it is whether the team that changes the reset flow can review, test, and roll back the exact email users receive.

This is an architecture decision record, not a ranking. The choice assumes that delivery status can be retrieved by polling, that the application can persist a cursor and normalized event IDs, and that password-reset messages share a recipient suppression policy with the game's other transactional email. If any of those assumptions is false, the boundary needs to move.

## How should custom password reset email polling own templates without webhooks?

The first invariant is ownership: the reset page, token semantics, email copy, localization, and release should have one accountable change path. Splitting the page across the game repository and the message across a provider dashboard creates a quiet deployment dependency. A copy edit can ship without the matching redirect, a locale can omit the recovery warning, or a rollback can restore code while leaving the live template untouched. None of those failures requires the mail API to be unavailable; they arise from two control planes disagreeing.

The second invariant is stricter. A recipient known to be invalid must not re-enter the send path just because another password-reset request arrives. The suppression decision therefore belongs in a durable data layer keyed by a normalized recipient digest, not in process memory and not only in a delivery-provider list. A provider-side suppression list may still be useful, but the application needs its own record because a provider migration must not erase accumulated delivery knowledge.

Delivery events are observations, not commands. The poller should translate provider-specific records into a small internal vocabulary such as `delivered`, `hard_bounce`, `soft_bounce`, and `complaint`; the policy layer then decides whether each observation suppresses future mail. This separation matters because a hard bounce and a transient deferral shouldn't have identical consequences, and because replaying an event must produce the same result. Don't let a raw status string write directly into the account table.

One more invariant is easy to miss: accepting an email for delivery is not evidence that it reached an inbox. The recovery endpoint can return a deliberately non-revealing response to the player while an internal delivery state progresses independently. That keeps account enumeration concerns out of the provider adapter and prevents a slow event poll from extending the interactive request.

No shortcuts.

There are three clocks. The player expects the recovery request to return quickly, the email system works on a delivery clock, and the poller discovers outcomes on its next successful pass. Treating them as one synchronous transaction produces bad retry behavior: a client timeout can cause a second send even though the first message was accepted, while a delayed bounce can arrive after the user has requested another reset.

The boundary is a local outbox. The account-recovery handler validates its own policy, creates a one-time recovery intent, and commits an outbox row in the same application transaction. A sender claims that row with an idempotency key and records the provider's opaque message identifier. Separately, the poller advances through delivery events and writes a normalized ledger. The next sender consults the ledger before sending. This design doesn't claim exactly-once delivery; it makes duplicate work observable and makes local state transitions idempotent.

The cursor is evidence.

Polling adds a specific uncertainty window. If the interval is 60 seconds, an invalid address can remain eligible until the bounce is observed, plus whatever delay exists before the event becomes visible. Sixty seconds is an example operating choice, not a delivery guarantee. A team should choose the interval from its recovery traffic, provider rate limits, and acceptable suppression lag, then record the measured lag as an operational metric. I'm not sure a universal interval exists; a load test and the selected API's documented quotas are what resolve that question. Persist the cursor only after all events in a page have committed, retain provider event IDs behind a uniqueness constraint, and overlap the query window when the API uses timestamps instead of an opaque cursor. That overlap intentionally fetches duplicates — the ledger absorbs them — and avoids losing a late-visible event at a page boundary. A malformed event goes to a quarantine record with its source ID and schema version; silently skipping it would make the cursor lie. Useful signals are mundane: oldest unsent outbox age, time since the last completed poll, event visibility lag, duplicate-event count, quarantined-event count, and attempted sends blocked by suppression. Alert on age rather than raw queue size because a busy but healthy launch day can create a large queue, while one stranded recovery email in a quiet region is still a correctness problem. Taken together, the cursor, unique event ID, quarantine record, and age metrics answer the question an operator actually faces after a restart: what evidence was durably interpreted, what evidence remains ambiguous, and how far the suppression view may lag reality?

## Where the reset message lives

The choice is not between "managed" and "custom" in the abstract. It is between release boundaries, and each boundary changes who can prove that the email matches the recovery flow.

| Option | Template owner | Strong fit | Failure boundary | Main limitation |
| --- | --- | --- | --- | --- |
| Authentication-managed template | Identity configuration owner | A small team wants one managed recovery flow and accepts dashboard-controlled releases | Identity configuration and mail delivery are coupled | Application code review does not contain the complete message change |
| Provider-hosted template | Messaging operations owner | Non-developers need controlled copy and localization changes | Provider template version can diverge from application deployment | Migration requires exporting and validating template behavior |
| Application-rendered template | Application team | Recovery behavior, copy, and game release must move together | Rendering errors and delivery submission are separate stages the team must operate | The team owns previewing, localization checks, and safe rendering |

For the stated gaming workflow, application-rendered templates win because template ownership is the primary decision axis. Put the subject and bodies in version control, render them before the provider boundary, and pass the adapter a complete message plus an idempotency key. The adapter should not know how a season name is localized or where the reset link points. It should know how to submit a message and how to expose events through a stable internal interface.

The catch is real: this choice is not suitable when the team cannot own HTML rendering, localization review, and mailbox testing. In that case, keep the template in the authentication system or a provider-hosted editor, and explicitly accept that a template release is a separate deployment. The correct owner is the group capable of testing and rolling back the artifact, not automatically the group writing Python.

Provider selection comes after that decision. Ask each candidate for documented answers to the same questions: Can events be listed without webhooks? Is pagination cursor-based or time-based? Are event IDs stable under repeated reads? How long are events retained? Which bounce categories are exposed? What authentication scopes allow read-only event polling? The API is disqualified if the answers cannot support replay, least privilege, and a bounded recovery procedure. A polished send endpoint cannot compensate for an event stream that cannot be reconciled.

## The suppression ledger in Python

The following code deliberately stops at a generic interface. `DeliveryClient` is the anti-corruption layer where a selected API's documented pagination and authentication behavior belongs; the policy below remains stable when that client changes. The normalized schema is internal, so the event names are design choices rather than claims about any vendor's payload.

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Iterable, Literal, Protocol


EventKind = Literal["delivered", "hard_bounce", "soft_bounce", "complaint"]


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    recipient_digest: str
    kind: EventKind


@dataclass(frozen=True)
class EventPage:
    events: tuple[DeliveryEvent, ...]
    next_cursor: str | None


class DeliveryClient(Protocol):
    def list_events(self, cursor: str | None) -> EventPage: ...


class Ledger(Protocol):
    def cursor(self) -> str | None: ...
    def has_event(self, event_id: str) -> bool: ...
    def record_event(self, event: DeliveryEvent) -> None: ...
    def suppress(self, recipient_digest: str, reason: str) -> None: ...
    def commit_page(self, next_cursor: str | None) -> None: ...


SUPPRESSING_EVENTS: frozenset[EventKind] = frozenset(
    {"hard_bounce", "complaint"}
)


def apply_event(ledger: Ledger, event: DeliveryEvent) -> None:
    if ledger.has_event(event.event_id):
        return

    ledger.record_event(event)
    if event.kind in SUPPRESSING_EVENTS:
        ledger.suppress(event.recipient_digest, reason=event.kind)


def poll_one_page(client: DeliveryClient, ledger: Ledger) -> int:
    page = client.list_events(ledger.cursor())
    for event in page.events:
        apply_event(ledger, event)

    # The implementation commits events, suppressions, and cursor atomically.
    ledger.commit_page(page.next_cursor)
    return len(page.events)
```

The production implementation needs one transaction around `record_event`, `suppress`, and `commit_page`; the protocol comment is not a substitute for that guarantee. It also needs a lease so two pollers can provide availability without racing the cursor, plus bounded retry with jitter for transport failures. Authentication secrets should be scoped to event reading where the selected service supports such scopes, and neither reset tokens nor raw addresses belong in logs. A keyed digest for recipient lookup is preferable to a plain hash because email addresses have a small, guessable domain.

Before deployment, test reordering, duplicates, an empty page, a page ending exactly at the cursor boundary, and a process stop after the last event write but before cursor commit. Then run the same fixtures against the real adapter's recorded, redacted responses. A schema change should quarantine an unknown record and hold the cursor rather than advance past evidence the policy has not interpreted.

## The boundary this ADR rejects

The rejected design sends the provider a stored template ID directly from the recovery handler and relies on the provider's suppression state. It is attractive because there are fewer local tables and the handler looks tiny. It fails this decision, however, because the application release no longer contains the message artifact, and a provider change would require reconstructing both template behavior and suppression history from outside the game's data layer.

Stick with that provider-owned design when messaging operations genuinely owns release approval, the provider's documented export and version controls meet the team's recovery objectives, and portability is less important than independent copy changes. It is also a reasonable boundary for a small system whose operators cannot support an outbox and reconciliation worker. Pretending a local poller is free would be worse engineering than accepting a documented dependency.

For this system, the final rule is compact: choose an API only after the team has assigned template ownership, proved replayable event retrieval without webhooks, and tested that an observed hard bounce blocks the next transactional send. Everything else is secondary.

## References

- https://postmarkapp.com/guides/transactional-email-best-practices
- https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
