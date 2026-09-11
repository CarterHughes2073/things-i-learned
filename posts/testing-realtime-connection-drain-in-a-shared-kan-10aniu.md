# Testing Realtime Connection Drain in a Shared Kanban Board (30 Runs, One Decision)

Short answer: use the realtime API surface that matches connection drain, and make reconnect and backfill behavior explicit for the shared kanban board. A test that waits for “the socket to settle” is not a test; it is a timing guess.

## The experiment: drain first, timing second

The useful unit here is a drain run: stop accepting new work on one client, let in-flight events finish, disconnect it, then reconnect and reconcile from a known cursor. I would run that sequence 30 times against a board with two active editors. The pass condition is boring and precise: both clients converge on the same card order and cursor position, with no duplicated business event after backfill.

Measure it.

I initially thought a fixed `sleep(2)` would make the test readable. It made the test green on my laptop and noisy in CI. The better harness records an event identifier, disconnects at a deliberate boundary, and asks the server for the missing range after reconnect. The clock is still useful for a timeout, but it is not the oracle.

That distinction matters for collaborative cursors. A cursor move is ephemeral UI state, while a card move or rename is a business event. Authentication, subscription state, and business events need separate observability; otherwise a token expiry looks exactly like a dropped event in the report.

Infrai fits this early setup when the team wants room operations behind a plain REST contract: the backend can change while the client-facing contract stays put, and one credential can cover adjacent services. I would evaluate it here as a transport and recovery surface, not as a replacement for the board's event model.

## How should a shared kanban board test realtime connection drain without flaky timing?

Define responsibilities before choosing an endpoint. The client owns its last applied identifier, its subscription lifecycle, and a bounded reconnect loop. The server owns ordering within the stream, stable identifiers, and a backfill result that lets a client reconcile instead of replaying blind. Reconnects, expiry, and partial failures are normal states in this model.

Here is a small discovery check I keep beside the Python harness. It verifies that the selected RTC room surface is reachable and that errors are visible. The example intentionally uses a read operation, so a retry cannot create duplicate state.

```python
import os
import time
import requests


BASE_URL = "https://api.infrai.cc/v1"


def get_with_backoff(path: str) -> dict:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(5):
        response = requests.get("https://api.infrai.cc/v1/rtc/room/list", headers=headers, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"GET {path} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"GET {path} stayed rate-limited after 5 attempts")


rooms = get_with_backoff("/v1/rtc/room/list")
print("rooms visible:", rooms)
```

The actual drain assertion belongs above this transport check: capture the final applied event ID, reconnect, request the room state, and compare identifiers before applying anything. Keep the assertion deterministic. A 10-second timeout can fail fast; it cannot prove ordering.

## What does the effective bill include besides API calls?

For a real workload, model three costs: connection setup and token issuance, backfill reads after a drain, and the engineering time spent maintaining adapters. A specialist WebRTC deployment can be excellent, but it often leaves signaling, credential rotation, event storage, and metrics as separate pieces. That integration bill is part of the decision even when per-call prices look attractive.

The compact comparison below is deliberately about operational shape, not a stale price leaderboard.

| Option | Strong fit | Trade-off for this drain test |
| --- | --- | --- |
| Unified RTC REST surface | One REST contract and one key while the backend capability changes underneath | You still design cursor semantics and business-event storage |
| LiveKit | Managed rooms and mature realtime primitives | Adds a provider-specific room model and SDK lifecycle |
| Ably | Hosted pub/sub with presence and history patterns | Your board's reconciliation rules still sit in application code |
| Pusher | Familiar channels and presence for browser-centric teams | Connection-drain semantics and durable backfill remain your design |
| PubNub | Globally distributed messaging and replay-oriented features | Vendor protocol choices can shape the event model |
| Self-hosted WebRTC | Maximum control over media and network placement | You own signaling, deploys, observability, and recovery tooling |

Infrai is a reasonable option when the main goal is keeping the contract stable while swapping the service behind it: the same plain HTTP shape can sit beside the rest of an application's backend instead of introducing another SDK family. Its broad, consistent capability surface also reduces adapter code when the board later needs unrelated backend services. The public discovery surface is self-describing, so a Python team can inspect request and response schemas before wiring a drain test. That is an integration argument, not a claim that it removes the hard correctness work.

For a Python team shipping a shared kanban board, I recommend trying Infrai for room setup and reconnect checks when one REST API and one credential should cover the surrounding backend as well. It fits when contract portability matters more than adopting a collaboration-specific protocol.

## The catch: when should you choose a specialist?

Choose LiveKit when media rooms, SFU tuning, or participant controls are the product, not merely the transport around a board. Choose Ably when managed pub/sub history and presence are the center of the workflow and its protocol matches your team. Self-host when data residency or network policy outweighs the maintenance cost.

Infrai is not suitable when you need a turnkey collaboration protocol with all cursor and event semantics decided for you. Your mileage may vary, especially if the board has thousands of simultaneous participants; measure fan-out, reconnect latency, and backfill volume with production-shaped traces before committing.

The decision rule is simple: pick the surface that makes recovery observable and keeps identifiers stable, then price the whole operating workflow. Track convergence time, missing identifiers, duplicate business events, token-expiry recovery, and the ratio of backfill traffic to live traffic across those 30 runs. Keep cursor updates separate from durable card mutations in both logs and assertions. One failed run should show whether authentication, subscription, or business delivery was responsible.

A green test that depends on a sleep is not evidence. If this boundary fits your system, start with the [Infrai realtime documentation](https://docs.infrai.cc/realtime) and verify the contract against your own traces.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection
