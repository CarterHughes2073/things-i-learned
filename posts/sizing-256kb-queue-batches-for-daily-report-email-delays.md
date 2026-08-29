# Sizing 256KB Queue Batches for Daily Report Email Delays

Short answer: for a property-management daily report email, schedule one run, store the report and recipient state outside the queue, and enqueue small reference-only batches that stay comfortably below 256KB; use delayed messages only when the requested send time is within seven days. This favors predictable recovery over the lowest possible request count. It also gives an AI feature team a clean place to measure generation latency and prompt cost before mail delivery starts.

Don't put the rendered report in the queue. A report can grow when a building gains units, when an agent includes more citations, or when a template changes. The job should identify what to load; durable storage should hold the content and delivery state.

## How can Python implement daily report email queue batches?

Start with the retry boundary. One report run represents a date and customer population. Within it, each active customer gets a delivery record with a stable identifier. The scheduler writes those records first, then publishes jobs containing only the run ID and delivery IDs. A worker loads the current records, verifies that each customer is still active, renders or retrieves the report, sends the email, and records the outcome.

Small batches trade a little more queue traffic for clearer retries and lower tail latency. Large batches reduce publish calls, but one slow render holds up every recipient behind it and a retry has more ambiguous partial work. There isn't a universal magic number. Begin with a conservative cap, measure serialized bytes and end-to-end latency, then change the cap using evidence from the actual property portfolio.

| Batch choice | Latency behavior | Cost and recovery consequence |
|---|---|---|
| One recipient | Independent progress | More queue operations; simplest retry scope |
| Small, bounded group | Slow work affects a few recipients | Balanced queue overhead; requires per-delivery state |
| Whole daily audience | Last recipient waits behind the entire run | Few publishes; widest partial-failure scope |

The 256KB message-size limit is a ceiling, not a target. Measure the encoded job immediately before publishing and reject it locally if it crosses an internal budget below that ceiling. The gap absorbs identifier growth and envelope changes. Never estimate size by character count: UTF-8 data and JSON encoding make that shortcut unreliable.

Measure bytes.

Delayed delivery answers a different question. It postpones an already-defined job by no more than seven days; it should not represent the recurring daily schedule. Cron is the recurring trigger. The distinction matters because the cron expression describes *when to create a run*, while the queue message describes *which run or deliveries to process*. Cron has several implementations, so confirm the exact expression and time-zone behavior of the scheduler you operate rather than assuming every cron dialect behaves identically.

This notebook-sized example creates deterministic delivery IDs, forms byte-bounded batches, and separates scheduling from delivery. The in-memory dictionaries stand in for a database and queue so the control flow stays visible. Production adapters should preserve the same contracts.

```python
import hashlib
import json
from dataclasses import dataclass
from datetime import date, datetime, timezone
from typing import Iterable


MAX_QUEUE_BYTES = 256 * 1024
INTERNAL_MESSAGE_BUDGET = 220 * 1024
MAX_RECIPIENTS_PER_BATCH = 100


@dataclass(frozen=True)
class Customer:
    customer_id: str
    property_id: str
    active: bool


def delivery_id(run_id: str, customer_id: str) -> str:
    raw = f"{run_id}:{customer_id}".encode("utf-8")
    return hashlib.sha256(raw).hexdigest()


def encoded_size(message: dict) -> int:
    body = json.dumps(message, separators=(",", ":"), ensure_ascii=False)
    return len(body.encode("utf-8"))


def make_message(run_id: str, ids: list[str]) -> dict:
    return {"run_id": run_id, "delivery_ids": ids, "schema_version": 1}


def build_batches(run_id: str, customers: Iterable[Customer]) -> list[dict]:
    messages: list[dict] = []
    current: list[str] = []

    for customer in customers:
        if not customer.active:
            continue

        candidate = current + [delivery_id(run_id, customer.customer_id)]
        candidate_message = make_message(run_id, candidate)
        too_many = len(candidate) > MAX_RECIPIENTS_PER_BATCH
        too_large = encoded_size(candidate_message) > INTERNAL_MESSAGE_BUDGET

        if current and (too_many or too_large):
            messages.append(make_message(run_id, current))
            current = [delivery_id(run_id, customer.customer_id)]
        else:
            current = candidate

    if current:
        messages.append(make_message(run_id, current))

    for message in messages:
        size = encoded_size(message)
        if size > MAX_QUEUE_BYTES:
            raise ValueError(f"queue message exceeds 256KB: {size} bytes")

    return messages


def create_daily_run(customers: list[Customer]) -> tuple[str, list[dict]]:
    reporting_date = date.today().isoformat()
    run_id = f"property-digest:{reporting_date}"
    messages = build_batches(run_id, customers)
    return run_id, messages


customers = [
    Customer("customer-1042", "building-17", True),
    Customer("customer-1043", "building-17", False),
    Customer("customer-2088", "building-29", True),
]
run_id, messages = create_daily_run(customers)
print(run_id, len(messages), encoded_size(messages[0]))
```

The fixed cap of 100 is a starting guardrail, not a performance claim. Run the same function against representative identifiers and then exercise the real worker in an evaluation harness. Track total run duration, p50 and p95 batch latency, queue age, messages per run, retries, recipients per batch, encoded bytes, report-generation tokens, and emails completed. If generation dominates, precompute a shared property summary and personalize from references. If provider calls dominate, controlled worker concurrency may matter more than a larger batch.

Here is the nasty case: a worker sends 37 emails from a 100-recipient batch and then loses its lease before recording the batch as complete. Retrying the entire batch without per-delivery state can duplicate those 37 messages; dropping the batch can omit the remaining 63. A batch-level `completed` flag can't describe the moment between the external send and the local write, either. Stable delivery IDs let the replacement worker inspect all 100 records independently, skip every completed ID, and continue only pending ones. The evaluation should interrupt execution after several different recipient positions, including immediately after the external side effect but before the state transition, because that narrow interval is where a tidy notebook often stops matching production. It should also rerun the daily scheduler and confirm that deterministic IDs prevent a second set of intentions from appearing. The batch is transport packaging, not the transaction boundary.

Fast enough wins.

## What should the failure-injection evaluation measure?

For active customers waiting on an operational digest, optimize the time from scheduled run to the last eligible delivery, not just average worker speed. A huge batch may look efficient in a request counter while worsening that last-recipient latency. Tiny batches increase queue operations and storage lookups. The useful decision rule is to choose the smallest batch that meets the run's cost budget without missing its delivery window, then validate it whenever recipient counts or report complexity shift.

This is where prompt-cost awareness belongs. Record model usage against the report run rather than hiding it in a worker log, and keep generated prose out of retry messages. A retry should point to a versioned artifact so it doesn't casually regenerate the report, consume more tokens, or produce a different summary. I'm not sure which batch size will minimize cost for an unseen workload; representative load tests, including the largest buildings and longest identifiers, are what resolve that uncertainty.

The catch is that reference-only jobs require durable application storage and reconciliation code. They are not suitable when the team cannot own a delivery state machine. In that case, choose an orchestration system that provides durable step state and inspect its documented message-size, delay, retry, and retention semantics before committing. Also avoid a seven-day delayed message when a business event may move repeatedly or farther into the future; persist the requested send time and let the scheduler release due work instead.

Backpressure must be explicit. Limit concurrent report generation separately from concurrent email sends, because their cost and latency profiles differ. Pause new fan-out when queue age breaches the delivery objective. Resume from delivery records, not from guesses based on how many jobs the scheduler says it created.

## Privacy governance for delayed messages

Observe the whole path with a run ID and delivery ID, but don't copy report text or unnecessary personal data into queue payloads and logs. GDPR Article 17 establishes a right to erasure in specified circumstances. A system still needs a defined retention policy and legal review, but centralizing report artifacts and customer mappings makes deletion materially easier than hunting through duplicated queue bodies. Delete or anonymize eligible records through one controlled workflow, while retaining only what another lawful obligation requires.

Seven days is also a governance boundary. A delivery requested beyond it belongs in durable scheduling state, where a customer status change or deletion request can be applied before any job is released. Even within seven days, the worker must recheck active status rather than treat the queued snapshot as permanent authorization to send.

No hidden payloads.

## Retry operations and the ship decision

Before shipping, run the pipeline twice for the same reporting date and confirm that it creates the same run and delivery IDs. Exercise inactive-customer changes between scheduling and execution. Test a job just under the internal byte budget, one just over it, a delay at the accepted boundary, and a request beyond seven days. Then interrupt a worker after partial progress and verify that replay completes only pending deliveries. These are evals for infrastructure behavior, and they belong beside prompt-quality evals in CI.

Ship when the duplicate-run test is clean, byte checks happen before publish, retries operate per delivery, and dashboards expose queue age plus last-recipient latency. Keep cron thin — it creates or releases work — and keep business truth in durable records. That architecture costs a few more lookups, yet its recovery story remains understandable when the daily report is late and the property team needs a precise answer.

## Sources

- https://en.wikipedia.org/wiki/Cron
- https://gdpr-info.eu/art-17-gdpr/
