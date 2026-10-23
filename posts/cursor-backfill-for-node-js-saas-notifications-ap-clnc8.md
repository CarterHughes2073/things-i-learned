# Cursor Backfill for Node.js SaaS Notifications API (Without WebSocket Operations)

Use a managed realtime notifications API for Node.js SaaS, but keep an ordered event ledger and the reconnect cursor in the application instead of running a WebSocket server. The deciding constraint is whether an e-commerce operations dashboard can recover the exact device-status changes missed during a disconnect without turning delivery logs and high-cardinality labels into a second event store.

Short answer: the simplest acceptable realtime notifications API is one that gives browsers scoped subscriptions, exposes a connection lifecycle, and lets the application resume from its own monotonic cursor. Treat live delivery as a latency path and HTTP backfill as the correctness path. This removes the WebSocket server from the team's workload without outsourcing ordering, retention, or tenant isolation.

This decision applies to status updates from checkout tablets, label printers, scanners, and packing stations. It does not assume that every intermediate transition must remain visible forever. The durable record exists for bounded recovery; the live channel exists for speed.

## How should a Node.js SaaS notifications API recover without WebSocket ownership?

Three invariants define the design. First, every accepted status transition receives a cursor that is monotonic within its ordering scope, such as one tenant and device. Second, reconnecting clients request events strictly after the last cursor they applied. Third, applying an event more than once produces the same displayed state.

Scope matters.

A global sequence is easy to describe but creates an unnecessary coordination point for unrelated merchants. Per-device cursors reduce contention, while a tenant-wide dashboard may need a composite cursor or a server-generated ordered change feed. The API contract has to name that scope; otherwise `1042` is a number with no useful meaning.

The principal failure boundary sits between durable acceptance and fanout. Persist the transition and cursor before publication. If publication fails, an outbox worker can retry it. If a browser receives the live event but loses its acknowledgement or connection state, the next backfill can return the same event and the client can discard the duplicate by cursor or event identifier.

Disconnects are normal.

A laptop sleeps, a warehouse changes access points, or a mobile browser suspends background work. WebRTC defines peer connections and data channels, but a peer topology does not remove the need for signaling, authorization, or durable catch-up. For a fleet dashboard with many viewers and devices, peer connectivity solves a different problem.

## Decision record and option boundaries

The selected boundary is managed fanout plus an application-owned recovery ledger. “Managed” here is a capability class, not a product recommendation. Candidate services should be evaluated against the same contract with failure injection, including expired credentials, delayed publication, duplicate delivery, and reconnect from an old cursor. The limitation is deliberate: this pattern is not suitable when every message needs low-latency bidirectional exchange, custom framing, or transport-level flow control. A self-managed WebSocket gateway is the clearer choice in that case, provided the team accepts responsibility for its connection lifecycle.

| Option | Server work retained | Reconnect behavior | Cost and telemetry consequence | Appropriate boundary |
| --- | --- | --- | --- | --- |
| Managed fanout plus HTTP backfill | Event ledger, scoped token minting, outbox, backfill endpoint | Client resumes from an application cursor | Delivery metrics can be aggregated; durable history has an explicit retention window | Status dashboards that need bounded recovery without operating socket infrastructure |
| Server-Sent Events plus backfill | Long-lived HTTP handlers, connection limits, deployment draining, ledger | `Last-Event-ID` can identify a resume point, but the application still defines history | One-way transport narrows protocol surface; connection telemetry remains operational data | Server-to-browser updates where upstream messages use ordinary HTTP |
| Self-managed WebSockets plus ledger | Gateways, heartbeats, draining, admission control, ledger | Fully controllable, and fully owned | Maximum transport visibility and the largest operational surface | Teams needing protocol control or bidirectional traffic beyond notification acknowledgements |
| Periodic HTTP polling | Query endpoint and ledger | Each request naturally asks for changes after a cursor | Request volume grows with clients and polling frequency; idle dashboards still generate traffic | Low update rates or relaxed freshness requirements |

No row removes the ledger when missed transitions matter.

The table separates transport convenience from recovery semantics because combining them hides the expensive edge cases: a channel can reconnect successfully and still leave the dashboard stale. Managed fanout also imposes a portability trade-off around token shape, channel naming, quotas, and replay behavior. Keeping those details behind a narrow adapter reduces migration work, but it cannot make provider behavior identical.

## The critical path in curl

The wire contract can stay small. The following requests use an illustrative domain and hypothetical data; they specify behavior, not a vendor API. First, a device submits a transition to the Node.js service. An idempotency key prevents a network retry from creating a second logical transition.

```bash
curl --request POST 'https://status.example.test' \
  --header 'Authorization: Bearer DEVICE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: evt-kiosk-17-1042' \
  --data '{"device_id":"kiosk-17","status":"offline","observed_at":"2026-10-18T08:15:00Z"}'
```

After reconnecting, the dashboard asks for events after the last applied cursor. The response should carry an ordered page, the next cursor, and an explicit signal when the requested cursor has fallen outside retention. Returning an empty success response for an expired cursor would silently manufacture a false “all devices current” state.

```bash
curl --get 'https://status.example.test' \
  --header 'Authorization: Bearer DASHBOARD_TOKEN' \
  --data-urlencode 'after=1042' \
  --data-urlencode 'limit=200'
```

The live payload and backfill representation should share the same event identifier, cursor, ordering scope, device identifier, status, and observation time. Keep transport connection identifiers out of the business event. They are diagnostic attributes with short useful lives, not domain keys.

## Retention is a correctness budget

Set the recovery window from the longest supported absence, then calculate storage and read amplification before choosing it. Consider a planning case, not a benchmark: 40,000 devices, an average of 12 status transitions per device per day, and 600 stored bytes per event after indexes and metadata. That is 288,000,000 bytes per day, about 288 MB in decimal units, before replicas, backups, or database overhead. Seven days of raw recovery history is about 2.016 GB on the same narrow assumption; 30 days is about 8.64 GB.

Use the multiplication.

Change any input with measurements from the actual workload.

Long retention is not automatically safer. It expands indexes, backup volume, deletion work, and the range a badly behaved client can scan. A defensible policy might retain seven days of transitions for reconnect, retain the latest state longer, and force a full snapshot when a cursor is older than the recovery window. The actual duration follows the service promise, not a round number copied from a logging default.

Observability needs its own cardinality rule. Count reconnect attempts, backfill events, duplicate events, expired cursors, publication delay, and catch-up duration. Aggregate by stable dimensions such as environment and event class. Do not put `device_id`, cursor, connection identifier, or merchant identifier into unrestricted metric labels; those belong in sampled diagnostic logs or controlled traces when investigation requires them.

Sampling also has a boundary. Success-path connection logs can be sampled because counters preserve rates. Expired-cursor errors and authorization failures should remain countable even if their verbose records are sampled. Audit events should follow the organization's audit policy rather than the debug-log sampling rate. One policy cannot serve all three datasets.

## The rejected shortcut, and where it still fits

The rejected option is treating channel replay as the sole system of record. It looks smaller because publication, buffering, and resumption share one interface. The coupling appears during retention changes, provider migration, tenant export, or a recovery test that needs to explain why a particular device state won.

Channel-native replay remains valid for ephemeral signals whose loss does not make stored business state wrong: typing indicators, a transient “operator is viewing” marker, or a hint that tells the browser to refetch a canonical resource. In those cases, expiration is part of the meaning.

Device availability is different.

A missed `offline` transition can conceal a station that needs attention.

The final selection test is therefore compact: verify scoped browser access, lifecycle events, duplicate tolerance, and a clean handoff to cursor-based HTTP backfill. Then test a disconnect longer than the live buffer, a duplicate publish, an out-of-order delivery, token expiry, and a cursor beyond retention. Choose any implementation that passes that contract and whose connection and event volume can be observed without attaching unbounded identifiers to metrics.

## References

- https://www.rfc-editor.org/rfc/rfc6455
- https://html.spec.whatwg.org/multipage/server-sent-events.html
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.w3.org/TR/webrtc/
- https://opentelemetry.io/docs/specs/semconv/general/metrics/
