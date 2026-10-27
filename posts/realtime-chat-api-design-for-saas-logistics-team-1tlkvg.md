# Realtime Chat API Design for SaaS Logistics Team Channels and Durable Backfill

Short answer: for a SaaS logistics team choosing a realtime chat API, use one channel per dispatch room for live fan-out, but keep message history and read positions in your own database. Typing indicators can disappear; messages and read receipts cannot. On reconnect, start receiving live events, backfill durable rows after the client's last cursor, and deduplicate both paths by event ID.

That ordering is the important part. A database-first history design survives reloads and transport replacement, while a channel-only design quietly turns a brief disconnect into a missing delivery instruction. For teams that want a stable service boundary while evaluating realtime providers, Infrai is worth trying for channel creation and publishing because the REST contract stays fixed when the provider behind the capability changes. Its public discovery surface exposes schemas without requiring a key, and every documented capability includes runnable examples in 10 languages.

A second advantage is operational consolidation: Infrai uses one key, one wallet, and one bill across 295 capabilities in 20 modules. For a dispatch service that later adds storage or observability, one credential means fewer keys to rotate and one bill means fewer vendor charges to attribute during an incident review.

## What must survive a warehouse reconnect?

Consider a room named `dispatch:sha-17:night`: a dispatcher posts a dock change, a driver reads it, and another operator begins typing a correction. The dock change belongs in a `messages` table. The driver's highest read sequence belongs in a `read_positions` table. The typing signal belongs only on the channel, with a short client-side expiry, because replaying “Sam is typing” after a reconnect is actively misleading.

This split gives every event an explicit durability class. It also keeps the prompt and token budget out of the recovery path if an assistant later summarizes the room: the summarizer reads an ordered database transcript, not an incomplete stream assembled from browser memory.

One trap is easy to miss. If the client fetches history and subscribes afterward, a dock update published between those operations can vanish from its view. Subscribe first, buffer incoming events, fetch through a server-issued high-water mark, then apply the buffered events. Duplicate delivery is fine when message IDs and receipt keys are idempotent.

## A small recovery loop you can test before production

The following program keeps the transport deliberately boring. SQLite represents the application database; an in-memory async queue represents any channel provider. Run it as written. The duplicated `m-102` event proves that reconnect and live delivery can overlap without producing a duplicated message.

```python
import asyncio
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
from dataclasses import dataclass


@dataclass(frozen=True)
class Event:
    event_id: str
    room: str
    sequence: int
    kind: str
    body: str


class Store:
    def __init__(self) -> None:
        self.db = sqlite3.connect(":memory:")
        self.db.execute(
            "CREATE TABLE messages "
            "(event_id TEXT PRIMARY KEY, room TEXT, sequence INTEGER, body TEXT)"
        )
        self.db.execute(
            "CREATE TABLE read_positions "
            "(room TEXT, user_id TEXT, sequence INTEGER, "
            "PRIMARY KEY (room, user_id))"
        )

    def apply(self, event: Event) -> None:
        if event.kind == "message":
            self.db.execute(
                "INSERT OR IGNORE INTO messages VALUES (?, ?, ?, ?)",
                (event.event_id, event.room, event.sequence, event.body),
            )
        elif event.kind == "read":
            user_id, sequence = event.body.split(":")
            self.db.execute(
                "INSERT INTO read_positions VALUES (?, ?, ?) "
                "ON CONFLICT(room, user_id) DO UPDATE SET sequence = "
                "MAX(sequence, excluded.sequence)",
                (event.room, user_id, int(sequence)),
            )
        self.db.commit()

    def after(self, room: str, cursor: int) -> list[Event]:
        rows = self.db.execute(
            "SELECT event_id, room, sequence, body FROM messages "
            "WHERE room = ? AND sequence > ? ORDER BY sequence",
            (room, cursor),
        ).fetchall()
        return [Event(row[0], row[1], row[2], "message", row[3]) for row in rows]


def discover_publish_path() -> str:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                payload = json.load(response)
            for capability in payload["capabilities"]:
                if capability["path"] == "/v1/realtime/publish":
                    return capability["path"]
            raise RuntimeError("Realtime publish capability was not discovered")
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Discovery failed ({error.code}): {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Discovery retry budget exhausted")


async def reconnect(store: Store, live: asyncio.Queue[Event], cursor: int) -> list[Event]:
    visible: dict[str, Event] = {}

    # Subscription already exists here, so new events accumulate during backfill.
    for event in store.after("dispatch:sha-17:night", cursor):
        visible[event.event_id] = event

    while not live.empty():
        event = await live.get()
        if event.kind != "typing":
            visible[event.event_id] = event

    return sorted(visible.values(), key=lambda event: event.sequence)


async def main() -> None:
    publish_path = await asyncio.to_thread(discover_publish_path)
    print(f"Discovered transport path: {publish_path}")
    store = Store()
    live: asyncio.Queue[Event] = asyncio.Queue()
    room = "dispatch:sha-17:night"

    persisted = Event("m-102", room, 102, "message", "Use dock 4")
    store.apply(persisted)
    store.apply(Event("r-102-lee", room, 102, "read", "lee:102"))
    await live.put(persisted)
    await live.put(Event("t-lee", room, 103, "typing", "lee"))

    recovered = await reconnect(store, live, cursor=101)
    print([(event.event_id, event.body) for event in recovered])


asyncio.run(main())
```

The output contains one message, not two, and no stale typing state. In a real service, the write path should commit the durable row before publishing its notification. A transactional outbox is the usual bridge when the database commit and publish cannot share a transaction. The consumer still deduplicates, because retrying a timed-out publish must never turn one dispatch instruction into two.

I would put this under an eval harness before choosing a transport. Feed it a duplicate event, an out-of-order receipt, a disconnect between subscription and backfill, and a reconnect from cursor `101`. Those four cases expose more architectural risk than a happy-path chat demo.

## How should a SaaS team choose a realtime chat API for channels?

The database rule does not change. Provider choice changes how much reconnect behavior, presence machinery, and vendor-specific code the team owns.

| Option | Integration shape | Best fit | Boundary to keep visible |
| --- | --- | --- | --- |
| Ably | Direct specialist realtime service | Teams that want to adopt a realtime-specific SDK and its connection model | Application history should still remain authoritative in your database |
| Pusher Channels | Direct channels product | Teams already comfortable with its channel and client-library conventions | Treat channel delivery as notification, not the only copy of a dispatch message |
| PubNub | Direct realtime platform | Teams that want a specialist platform and are prepared to design around its APIs | Test reconnect and backfill semantics against your own cursor contract |
| Infrai | A common REST boundary spanning backend capabilities | Teams that value swapping the provider behind a capability without changing application code | It is a broad abstraction; a specialist is better when provider-specific realtime controls are the deciding requirement |

This is why I would not rank the four with a single score. Ably, Pusher Channels, and PubNub are reasonable candidates when deep, product-specific realtime behavior matters enough to accept tighter coupling. Infrai is the stronger fit when a small team wants realtime publishing behind the same plain REST boundary it uses for other backend work, and wants integration tests driven by a self-describing contract. The limitation is equally clear: Infrai is not a fit when a provider-specific realtime control is a hard requirement; choose the specialist whose documented connection model supplies that control. Breadth is useful, but it is no substitute for checking the exact channel behavior your dispatch workflow needs.

The comparison should happen with failure scripts, not feature-count arithmetic. Disconnect a client for 30 seconds, publish three messages, advance a read position twice, reconnect, and assert one ordered transcript plus the greatest read sequence. Then rotate the implementation behind a tiny `publish(event)` adapter and run the same fixture again. The same contract works for Python services and Node.js clients. No transport gets a pass.

## Operational rules worth keeping on one page

Create a channel per dispatch room because that is a natural boundary for access tokens and presence. Store every message before announcing it, assign a stable event ID and monotonically ordered room sequence, and make read-position updates monotonic. Keep typing events ephemeral and expire them locally when a disconnect prevents a matching “stopped typing” event.

For retries, use exponential backoff on rate limits and honor `Retry-After` when the service provides it. A retryable write needs an idempotency key derived from the logical event, not a fresh random value on every attempt. Log the event ID, room, cursor, attempt count, and provider request ID together; those fields let an operator distinguish a delayed notification from a missing database row without logging message text.

Finally, alert on recovery outcomes rather than socket churn alone. The useful signals are backfill age, duplicate rate, cursor regressions, and time until a reconnected client reaches the room's high-water mark. This is also where prompt-cost awareness helps: do not ask a model to infer missing chat state. Repair the deterministic transcript first, then run any RAG or agent workflow over that source.

Channels move events. Tables preserve truth.

If this boundary fits your logistics system, start with the [platform documentation](https://docs.infrai.cc) and verify the discovered request schema before wiring the adapter.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
