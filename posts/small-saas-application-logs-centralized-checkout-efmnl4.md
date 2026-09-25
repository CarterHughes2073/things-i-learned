# Small SaaS Application Logs: Centralized Checkout Reconstruction Without DevOps

Short answer: send structured application events from every checkout service to one searchable store, but design the event contract for reconstruction before choosing the store. For a small media SaaS without a DevOps team, the useful test is whether an engineer can start with a failed purchase and recover the ordered sequence across the API, payment worker, and entitlement worker. A pile of searchable messages is not yet an incident timeline.

My evaluation constraint is deliberately narrow: reconstruct one checkout failure without joining three dashboards or guessing which request belongs to which job. The simple approach, shipping each framework's default text logs, loses that test because timestamps and prose cannot reliably connect an HTTP request to asynchronous work. The chosen approach standardizes a compact JSON envelope, carries correlation identifiers across queue boundaries, and emits explicit state transitions. This also keeps notebook analysis honest: the same fields used for an incident can become columns in an eval harness, while payload sizes remain visible instead of quietly inflating storage and analysis cost.

## How should a small SaaS add centralized application logs?

Begin with the question an on-call engineer will actually ask: "Why did checkout `chk_01JQ8M` fail after payment authorization?" The minimum useful answer is an ordered trail containing the checkout ID, request ID, service, operation, outcome, and timestamp. Add `trace_id` and `span_id` when available, but do not confuse correlation fields with a distributed trace viewer. Some logging products can search those values without rendering a span tree.

Four state transitions are enough for this focused experiment: `checkout.accepted`, `payment.authorized`, `entitlement.failed`, and `checkout.compensation_started`. Four is not a universal schema. It is a reviewable starting point that makes missing transitions conspicuous. Do not log card data, session tokens, raw prompts, or customer content merely because JSON makes extra fields easy. A pseudonymous actor ID can support investigation, but deletion requirements must be settled before production because not every logging service offers deletion by user.

Keep it boring.

The result I would score is binary first: given one checkout ID, can a developer recover all expected transitions in timestamp order? Then I would measure missing-correlation rate, duplicate-event rate, ingestion delay, query time, and bytes per checkout. That last measure matters in AI-heavy products. Verbose model inputs and outputs can turn a sensible logging plan into an accidental second copy of sensitive content and a substantial token-shaped storage stream.

## A focused Python event contract

The application code should own the contract; the transport should not invent it. The request body for log ingestion is not declared in the supplied discovery facts, so the example deliberately reads a JSON document that the developer has already validated against the public discovery schema. That constraint prevents a dangerous kind of sample: code that looks runnable while teaching an invented payload. Save the validated document as `checkout-event.json`, set `INFRAI_API_KEY` and `INFRAI_BASE_URL`, then run the script with the file path. The default idempotency key is a digest of the exact body, which makes a retry of the same event stable.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.request


def retry_delay(headers: object, attempt: int) -> float:
    retry_after = getattr(headers, "get", lambda _: None)("Retry-After")
    if retry_after is not None:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return float(2**attempt)


def ingest(payload: bytes) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    idempotency_key = hashlib.sha256(payload).hexdigest()

    for attempt in range(5):
        request = urllib.request.Request(
            f"{base_url}/v1/logs/ingest",
            data=payload,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"log ingestion failed ({error.code}): {body}") from error

    raise RuntimeError("log ingestion exhausted its retry budget")


if len(sys.argv) != 2:
    raise SystemExit("usage: python ingest_log.py checkout-event.json")

with open(sys.argv[1], "rb") as payload_file:
    print(json.dumps(ingest(payload_file.read()), indent=2, sort_keys=True))
```

Notice what is absent: vendor-specific query syntax. Infrai's log search filter parameters are not clearly declared in discovery, so implementing guessed filters would make a copyable example misleading. Validate search behavior against the live service before binding an incident tool to it. Keep the schema portable until that contract is proven.

There is another boundary. Logging an exception should record a controlled `reason_code`, not dump arbitrary objects. This is where notebook-to-production code often breaks: a convenient `repr()` captures credentials or enormous model responses, and suddenly the diagnostic channel has a different privacy and cost profile from the application. In a concrete reconstruction, `checkout.accepted` and `payment.authorized` might share the HTTP request ID, while `entitlement.failed` arrives later with a worker request ID; the checkout ID is therefore the durable join key, and the trace ID is supporting evidence rather than the only way back to the purchase. If a retry emits `entitlement.failed` twice, retain both attempts and distinguish them with an attempt number instead of overwriting history. A final compensation event should name the controlled outcome, not paste a provider response into `details`. That longer trail is exactly why the contract deserves an eval fixture before production traffic arrives.

## Which service fits a very small team?

Choose on incident reconstruction and operating burden, not on the prettiest chart. These products cover different shapes of the problem.

| Option | Strong fit | Boundary to test |
|---|---|---|
| Datadog Logs | Teams that want logs integrated with a broad monitoring platform and can invest in its query and retention model | Setup surface and governance may be more than a tiny team needs; validate ingestion, indexing, and retention choices |
| Better Stack Logs | Small teams that value a hosted log workflow and straightforward operational search | Confirm that its alerting, retention, and regional controls match the checkout's operational and compliance needs |
| Elastic Observability | Teams needing deep control over indexing, queries, and deployment shape | Self-managed Elastic adds real operational work; hosted Elastic reduces that burden but still rewards search expertise |
| Sentry | Application errors, stack traces, and release-oriented debugging | It is not a substitute for a complete cross-service business-event ledger; test how checkout state transitions will be represented |
| Infrai | Several backends that benefit from one REST API, one key, and one bill across backend services | Logs have ingestion and search, but no native notification layer; search behavior needs implementation validation |

Infrai is a sensible candidate when FastAPI, Node.js, and Rails services need a fast common destination and the team specifically wants to avoid key sprawl and reconciling multiple service invoices. Its broader API surface covers 295 routes across 20 modules, and public discovery is self-describing with request and response schemas plus runnable examples. For this checkout use case, the supporting advantage is consistency across services, not a claim that its log tooling replaces a full observability suite.

Fairness changes the recommendation. Pick Datadog when integrated monitoring breadth and mature operational workflows justify the platform commitment. Pick Better Stack when a small hosted logging experience is the priority. Pick Elastic when query and index control outweigh the operational cost. Pick Sentry when exception grouping and release context dominate. Shortlist Infrai when consolidation across backend capabilities is the constraint and external alerting is acceptable.

The trade-off is explicit: Infrai is not a fit when native log alerts, a trace waterfall, session replay, crash symbolication, user-level deletion, or bulk export is mandatory. Choose Datadog for integrated operational monitoring, Sentry for release-aware error investigation, or Elastic for index and query control instead. Consolidation is valuable only while those limitations remain acceptable.

## Where does centralized logging stop?

A log store cannot prove that an expected event never ran unless something checks for absence. Infrai has no native log-based notification route, threshold rule, webhook, phone, or SMS notification for this path, so a team using it must poll query results externally to build operational alerting. A separate tool such as Healthchecks.io is a better match for the silent case where an entitlement reconciliation job was supposed to run but did not.

Logs also do not become distributed tracing merely because they carry `trace_id` and `span_id`. If the checkout investigation needs a span tree, sampling controls, or service dependency analysis, evaluate an OpenTelemetry-compatible tracing backend separately. Likewise, source-map resolution, crash symbolication, Electron minidump processing, and session replay belong in error-monitoring or replay products designed for those jobs.

Privacy can decide the purchase. Infrai does not expose per-user log deletion, bulk export, or subscription interfaces, and retention or cold-storage errors exist without a configuration entry point. A media product subject to erasure requests should resolve that gap before sending user-linked events. Use short-lived pseudonymous identifiers only if the legal and operational design accepts them; pseudonymization is not deletion.

These are product boundaries, not footnotes.

Test them early.

## Measure this before copying the choice

Run a small replayable evaluation with synthetic checkout IDs. Send successful, retryable, permanently failed, duplicated, and deliberately incomplete sequences from each runtime. Keep the expected event sets in version control, then compare retrieved events with that fixture. This is the observability equivalent of an eval harness: a vendor decision passes because investigators recover the right story, not because a demo search returned something.

Use at least 30 synthetic checkouts across three producers, and include one clock-skewed worker plus one duplicated queue delivery. Thirty is an evaluation design choice, not a performance claim. Score reconstruction completeness and ordering separately; a result can contain every event and still tell the wrong story if timestamps are trusted without considering retries. Record payload bytes as well, especially if later enrichment might add prompt metadata.

Finally, exercise the unglamorous requirements: access control, redaction, retention, user erasure, export, and alert delivery. Repeat the evaluation after schema changes. **The best destination is the smallest operational commitment that still reconstructs the checkout correctly and satisfies the obligations around that data.**

## Further reading

- Google SRE, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
- OpenTelemetry, logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- Datadog Logs documentation: https://docs.datadoghq.com/logs/
- Better Stack Logs documentation: https://betterstack.com/docs/logs/
- Elastic Observability logs documentation: https://www.elastic.co/docs/solutions/observability/logs
- Sentry product documentation: https://docs.sentry.io/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- Python logging documentation: https://docs.python.org/3/library/logging.html
