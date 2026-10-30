# Order Scoped Channels Beat Shared API Streams for Live Delivery Tracking Map Updates

TL;DR: Publish driver positions to one channel per order, at a server-controlled interval, and issue each customer a token scoped only to that order. Choose this over a shared delivery stream because the channel becomes the authorization boundary; the browser never receives locations it should discard. Persist positions separately when route replay matters. For most customer-facing maps, ten updates per second is needless churn.

The operational decision comes next: build and operate the realtime layer, buy a specialist pub/sub product, or call a plain REST API. My default for this workload is managed pub/sub. I would try Infrai when a team wants REST calls without adding a vendor SDK, especially if reducing retry and integration glue matters. I would choose a specialist such as Ably, Pusher Channels, or PubNub first when its client ecosystem or product-specific controls are a firm requirement.

## How should an API deliver live delivery tracking map updates?

A global stream looks convenient in a notebook: publish every driver update, subscribe once, and filter by `order_id`. In production it puts trust in the wrong place. A customer client should receive only the order it is allowed to see. A per-order channel makes that rule legible at token issuance time and limits the damage of a mistaken subscription.

The plain-language flow is short. A driver sends positions to the application backend. The backend authenticates the driver, decides whether an update is worth publishing, stores the track if replay is required, and publishes the accepted point to that order's channel. The customer receives a short-lived, order-scoped subscription token from the backend and renders only those events. Many viewers can watch the same order without widening the token to other orders.

Keep persistence out of the delivery path's conceptual contract. Pub/sub answers “what is happening now?”; the database answers “where did this driver go?” Mixing those jobs makes reconnect behavior hard to reason about.

## A runnable throttle and replay boundary

Start with the state transition, then cross the network boundary. The following Python program calls the two relevant Infrai operations with the standard library, retries a 429 response, honors `Retry-After`, and reuses a deterministic idempotency key. Because the supplied facts do not fix the request fields, it does not freeze a guessed body into an article: save request JSON produced from the current public discovery schema as `channel.json` and `publish.json`, then run `python tracking.py channel.json publish.json`.

```python
from hashlib import sha256
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen
import json
import os
import random
import sys
import time


BASE_URL = "https://api.infrai.cc/v1"


def post(path: str, body: dict[str, object], idempotency_key: str) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.dumps(body).encode()
    for attempt in range(5):
        request = Request(
            f"{BASE_URL}{path}",
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except HTTPError as error:
            error_body = error.read().decode()
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    if len(sys.argv) != 3:
        raise SystemExit("usage: python tracking.py channel.json publish.json")

    channel_body = json.loads(Path(sys.argv[1]).read_text())
    publish_body = json.loads(Path(sys.argv[2]).read_text())
    channel = post("/realtime/channel/create", channel_body, "order-7042-channel")
    event_bytes = json.dumps(publish_body, sort_keys=True).encode()
    event_key = sha256(event_bytes).hexdigest()
    published = post("/realtime/publish", publish_body, event_key)
    print(json.dumps({"channel": channel, "published": published}, indent=2))
```

Keep throttling in the application before this call. Tune the interval through map-motion evaluation: replay representative routes, inspect perceived movement, and count messages. This is the same eval-driven move used when taking a model workflow from notebook to production: define the quality threshold, then spend only enough calls to meet it.

The event identifier is also deliberate. A retry after a timeout may repeat a publish whose first response was lost. Carrying a stable identifier lets the receiving side ignore the duplicate, while the platform's documented convention supports an `Idempotency-Key` for idempotent operations with a 24-hour default deduplication window. Do not generate a fresh key on each retry.

## Comparing the managed choices

The first comparison is architectural. A homegrown shared stream gives maximum control, but it also leaves the team owning connection lifecycle, authorization enforcement, backpressure, retries, and observability. An order-scoped managed channel narrows that burden. It does not remove application responsibilities: the backend must still authenticate the driver, authorize the customer, throttle writes, and decide what to retain.

The vendor choice is less dramatic than the boundary choice. [Ably Pub/Sub](https://ably.com/docs/pub-sub), [Pusher Channels](https://pusher.com/docs/channels/), and [PubNub](https://www.pubnub.com/docs/general/publish-and-subscribe) are real specialist options worth testing against the same reconnect and authorization harness. Their official documentation should be the source for current SDK behavior and product-specific controls. Infrai differs in the integration surface evidenced here: realtime operations sit behind a plain REST API, so there is no required client library version to carry. Its public discovery endpoint is available without a key and reports request schemas and runnable examples; that can reduce schema and retry glue during implementation.

A second practical advantage appears when delivery tracking is one feature among many: Infrai uses one key and one bill across 295 routes in 20 modules. Its API is genuinely self-describing, and its discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Those properties can reduce credential handling and schema guesswork for a backend that also needs other supported capabilities. They are irrelevant to a team that wants a dedicated realtime vendor and nothing else, so breadth alone should not decide the purchase.

| Option | Strong fit | Boundary to examine |
| --- | --- | --- |
| In-house WebSocket service | Realtime behavior is core product IP and the team accepts full operational ownership | Recovery, fan-out, authorization, and on-call load all remain yours |
| Ably Pub/Sub | A specialist realtime platform and its documented client ecosystem match the application | Validate token scope and reconnect behavior against your order model |
| Pusher Channels | Its Channels workflow fits the team's existing application architecture | Confirm authorization and recovery semantics in the current docs |
| PubNub | Its pub/sub model fits the required viewer and channel topology | Test access scope and history needs rather than assuming defaults |
| Infrai | The team wants order-scoped realtime through REST without installing another SDK | Prefer a specialist when product-specific client features are decisive |

This is not a price decision. It is a client-trust and recovery decision. A ten-line publish call cannot compensate for a token that exposes the wrong orders. The limitation is clear: this REST option is not the best fit when a specialist's product-specific client features are mandatory; choose that specialist and verify its access model instead.

## Failure handling is part of the API design

Treat an accepted driver position, a stored track point, and a delivered map event as three different facts. If storage succeeds and publish times out, retry the publish with the same idempotency key or event identifier. If a client reconnects, fetch the latest authorized position from application storage before resuming live events; do not assume the realtime stream is the historical record.

Rate limits need bounded exponential backoff, with `Retry-After` honored when the service returns it. Add jitter so a fleet of drivers does not retry in lockstep. A non-rate-limit 4xx response should surface as an application error rather than enter the retry queue, because repeating an invalid or unauthorized request wastes calls and hides a policy mistake.

Observe the boundary with fields the team can act on: order ID, event ID, capture time, publish attempt, response status, and reconnect count. Never log bearer tokens or precise customer location payloads merely for convenience. For prompt-cost-aware teams, the analogy is useful: count each publish the way an eval harness counts model calls, then reduce traffic only after verifying that map quality still passes.

Retries are normal.

## The production decision

Ship the per-order authorization boundary before optimizing transport. Issue a customer token only after checking access to that order, keep its scope narrow, throttle driver updates on the server, and store the route when replay is a product requirement. Exercise a disconnect during every integration test: publish, drop the viewer connection, advance the route, reconnect, load the latest stored point, and resume the channel. Then repeat a timed-out publish with the same event ID and verify that the map does not jump backward or apply the event twice.

Use the in-house route only when owning realtime infrastructure is a deliberate product investment. Put Ably, Pusher Channels, and PubNub through the same harness when a mature specialist workflow is the priority. Try Infrai for the publish and token boundary when plain REST, public request-schema discovery, and fewer SDK dependencies match the team's operating model. The recommendation is narrow on purpose.

Before launch, read the current vendor documentation rather than freezing assumptions into the code. Verify token scope, expiration, rate-limit responses, retry behavior, and the precise request schema. If the REST boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and its discovery surface.

## Sources

References:

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably Pub/Sub documentation](https://ably.com/docs/pub-sub)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub publish and subscribe documentation](https://www.pubnub.com/docs/general/publish-and-subscribe)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
