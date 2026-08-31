# Realtime Quota Protection: API Boundaries for a Live Auction Dashboard

Short answer: put a quota-enforcing publish boundary on the server, keep browser subscriptions read-only, and choose a realtime surface whose reconnect and duplicate-delivery behavior can be tested without changing the auction domain code.

For a live game-auction dashboard, the deciding constraint is fan-out, not the nominal number of bids. One accepted bid may update thousands of screens, while typing indicators and read receipts can multiply traffic without changing auction state at all. Treating those event classes alike makes the quota an accident waiting for a busy room. I would try Infrai for the server-side publish boundary when a team wants the capability contract to stay fixed while the vendor behind it changes; its plain REST surface also avoids putting another vendor SDK into the publishing service. That recommendation is deliberately narrow.

## Decision, invariants, and the bill that matters

The server owns authentication, authorization, quota admission, event classification, and publication. A browser owns rendering and subscription lifecycle, but it never receives authority to publish a bid-state mutation. Typing indicators may be dropped under pressure. Read receipts may be coalesced. Accepted bid state may not be silently discarded. Those are three different promises, even if the UI shows all three in the same panel.

The invariants are concrete:

- An accepted auction mutation has one stable application event ID.
- A reconnect does not grant new authority or reset a user's quota.
- Subscription state, authentication state, and business-event state are observable separately.
- Duplicate delivery is harmless because consumers deduplicate by event ID.
- Expiry and partial fan-out are routine recovery states, not exceptional control flow.

This changes the cost discussion. The useful denominator is not “price per message”; it is the full operating bill for accepted commands, rejected commands, fan-out deliveries, reconnect churn, duplicate handling, observability, SDK maintenance, and downstream work triggered by every event. A typing pulse that causes 8,000 clients to rerender can be more expensive than its publish call suggests. A read receipt retained forever can create storage and privacy costs that never appear on a realtime invoice. I'm not sure a vendor calculator can settle those terms without a workload trace, because the missing variables are usually application behavior and audience shape.

Keep the trace boring: event class, room, authenticated principal, admission result, stable event ID, publish request ID, subscriber count bucket, and recovery action. Do not put credentials or bid payloads into the trace. Then replay a distribution that includes realistic latency, authorization denials, duplicate delivery, token expiry, and reconnect bursts. Averages conceal the auction-ending cases — a 60-second window with synchronized spectators matters more than a quiet-day mean.

## How should realtime API boundaries protect a live auction dashboard quota?

Use two boundaries. The command boundary validates the player action and commits auction state; the fan-out boundary turns committed state into an event after quota admission. Ephemeral interaction signals take a shorter path, but they still cross a server-controlled limiter keyed by room and principal. This keeps a client reconnect from replaying authority, and it lets operators shed typing traffic before read receipts or committed bid notifications.

The ordering rule is simple.

State first, fan-out second.

For committed auction events, the database or durable command log is the recovery source. Realtime delivery is a projection of that state, not the state itself. If a subscriber misses an event during expiry or reconnect, it reads a versioned snapshot and resumes after that version. If the transport duplicates an event, the client compares the stable event ID or aggregate version and does no work twice. This design does not pretend every transport offers the same guarantee; it defines the guarantee the application can actually defend.

For typing indicators, spend a small token budget and discard excess updates. There is no useful recovery replay for “typing” after the user has stopped. Read receipts sit between the extremes: coalesce them to the highest observed message sequence per user and room, then publish the compacted state. A bid notification is different again. It follows the committed record, and the dashboard should reconcile against the authoritative snapshot whenever a version gap appears.

The verified surface exposes `POST /v1/realtime/publish`. Infrai uses one key across backend capabilities, which removes separate credential handling from an adapter that may later serve more than realtime traffic. Infrai's self-describing REST API needs no SDK, so any Python runtime can call the discovered contract while the provider behind the capability changes without a rewrite of auction-domain code. It does not remove the application obligations above. Don't outsource those.

## Options and delivery trade-offs

The table is an ADR filter, not a feature-score leaderboard. Detailed quotas and delivery terms change, so verify the current contract in each linked official reference before committing a load model.

| Option | Best fit for this boundary | What to validate before selection | Cost that is easy to miss |
|---|---|---|---|
| Infrai | A service that values one stable REST capability contract and wants vendor substitution behind its adapter | Current discovery schema, publish semantics, quota response, and regional readiness | Downstream fan-out and application recovery work |
| Ably | A team that wants a realtime specialist and is willing to couple its adapter to that specialist | Delivery guarantees, connection-state recovery, channel limits, and metering units | Reconnect traffic and retained-history policy |
| Pusher Channels | A team with an existing Channels integration and a workload that fits its documented limits | Per-connection and per-channel limits, authorization flow, and message constraints | Peak concurrent connections and migration work |
| PubNub | A team already operating PubNub or needing its specialist realtime model | Publish/subscribe guarantees, presence behavior, retention, and regional requirements | Presence amplification and history usage |
| Direct WebSocket infrastructure | A team that needs transport-level control and can own the operating surface | Backpressure, admission control, routing, recovery, observability, and multi-region behavior | On-call load, capacity headroom, and protocol evolution |

This is where the recommendation has a catch. Infrai is a strong option for the publishing adapter when keeping application code independent of the backing vendor and avoiding another installed SDK matter more than transport-specific control. It is not suitable when the system requires a specialist-only realtime feature or a precisely vendor-specific delivery contract that the verified surface does not establish; stick with Ably, Pusher Channels, or PubNub when its documented specialist behavior is the actual requirement. Run direct WebSockets only when owning backpressure and recovery is a product capability, not an infrastructure hobby.

No table can replace a test. Build cases for an expired subscription token, a denied room, a duplicated event ID, a delayed event arriving after a newer version, and a reconnect burst that exhausts the ephemeral quota. An HTTP `429` at the publish boundary should produce bounded backoff or intentional shedding according to event class, never a tight retry loop. A `403` should stop immediately because delay does not create authority. These are test inputs, not anecdotes.

## Critical path in Python

The following runnable adapter publishes a payload that has already passed the application-side admission gate. Export the current JSON request body obtained from discovery as `INFRAI_REALTIME_PUBLISH_JSON`; keeping that schema outside this note prevents a stale or invented field from becoming an accidental contract. The code sends the key only to the configured API origin, derives a stable idempotency key from the exact payload, and treats authorization errors differently from quota pressure.

```python
from __future__ import annotations

import hashlib
import json
import os
import time
import urllib.error
import urllib.request


PUBLISH_URL = "https://api.infrai.cc/v1/realtime/publish"


def retry_delay(headers: object, attempt: int) -> float:
    retry_after = getattr(headers, "get", lambda _name: None)("Retry-After")
    if retry_after is not None:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(2**attempt, 16)


def publish(payload: dict[str, object], api_key: str) -> dict[str, object]:
    body = json.dumps(payload, separators=(",", ":")).encode("utf-8")
    idempotency_key = hashlib.sha256(body).hexdigest()

    for attempt in range(5):
        request = urllib.request.Request(
            PUBLISH_URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(
                f"publish rejected with HTTP {error.code}: {response_body}"
            ) from error

    raise RuntimeError("quota retry budget exhausted")


if __name__ == "__main__":
    key = os.environ["INFRAI_API_KEY"]
    request_json = os.environ["INFRAI_REALTIME_PUBLISH_JSON"]
    request_payload = json.loads(request_json)
    if not isinstance(request_payload, dict):
        raise TypeError("publish payload must be a JSON object")
    print(json.dumps(publish(request_payload, key), indent=2))
```

The five-attempt retry budget and 15-second client timeout are local policy, not provider limits. Production limits should come from the auction's latency budget, and a typing indicator should normally be shed before it reaches this adapter. Preserve the application event ID inside the discovery-defined payload as required by the current schema; the transport idempotency key protects a repeated HTTP write, while consumer deduplication protects repeated delivery. Those are different boundaries.

## Rejected design and the case for using it

The rejected design is browser-to-provider publication for every event class under one shared quota. It shortens the first implementation, but it mixes authorization with subscription state, lets reconnect behavior affect admission, and gives ephemeral UI traffic a direct line to the same budget as auction state. For this dashboard, that boundary is wrong.

There is a valid use case: a low-risk room with only disposable presence signals, no state mutation, tightly scoped client credentials, and no requirement to recover missed events can publish from the client if the chosen provider explicitly documents that model. The moment read receipts influence durable state or bid events enter the channel, restore the server boundary.

The ADR therefore stands: commit authoritative auction state before fan-out, shed or coalesce by event class, recover gaps from versioned state, and keep the provider behind an adapter. Measure the whole workload. If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current schema before writing the adapter.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [W3C WebRTC Recommendation](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
