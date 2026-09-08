# Node.js Service Design for 2,000 HR Onboarding Packets with Asynchronous PDF Jobs

To implement HR onboarding packets, a Node.js service should treat asynchronous PDF jobs as a batch system, with explicit validation and cleanup before it renders and archives 2,000 packets in one monthly run.

Short answer: choose explicit asynchronous PDF jobs with bounded retries, preflight validation, private temporary storage, and deterministic manifests. A synchronous merge endpoint is fine for a small interactive packet, but it becomes the wrong control loop when latency and throughput are the decision axis.

I started with the simple design: upload a few files, call a merge operation, wait for the response, and write the result to the archive. It is pleasant in a notebook. Under load, it ties a worker to the slowest document, makes retries ambiguous, and leaves no durable explanation for why packet 1,437 differs from packet 1,438. The better design treats each packet as a job with a correlation ID and a state transition that can be audited later.

## How should a service handle asynchronous PDF jobs, retries, validation, and secure temporary files?

Start with a cheap gate before any job leaves the service. Check the MIME type, page count, and byte size of every input. Reject a renamed executable that says `application/pdf`, reject a packet above the operational page limit, and reject a file that cannot be parsed. These checks protect throughput as much as they protect security: a bad input should not consume a queue slot or a PDF worker.

Create one correlation ID per packet, and carry it through the job record, logs, output object, and manifest. The worker submits the PDF operation, then polls `GET /v1/pdf/job/get/{job_id}` with bounded exponential backoff. A practical sequence is 1, 2, 4, 8, and 16 seconds, capped by a deadline selected from the monthly batch SLO. The cap is the important part. An unbounded poller quietly turns a vendor delay into a growing fleet of occupied workers.

Retries need two separate policies. Network timeouts and a 429 are retryable; a validation rejection is not. For a retryable response, honor `Retry-After` when it is present, otherwise back off and add a small random jitter. Give the submission an idempotency key derived from the correlation ID and packet version so a process restart cannot create two archive documents. Consumers still need idempotency because a standard queue is at-least-once by nature.

Temporary files are inputs, not archives. Put them in a private, signed-only location, keep output objects in a separate prefix, and delete the input artifacts after the final manifest is committed. Do not pass an authorization header to a returned presigned URL. The archive should contain the output reference and retention policy, while the staging area should have a short, explicit lifecycle.

That is the whole trick.

Here is the local preflight shape I use before enqueueing. It is intentionally boring; boring checks are cheap checks.

```python
import os
import random
import time
from pathlib import Path

import requests
from pypdf import PdfReader


def validate_pdf(path: str, max_bytes: int, max_pages: int) -> dict:
    file_path = Path(path)
    if file_path.stat().st_size > max_bytes:
        raise ValueError("file exceeds the packet size limit")

    header = file_path.read_bytes()[:5]
    if header != b"%PDF-":
        raise ValueError("file is not a PDF")

    pages = len(PdfReader(str(file_path)).pages)
    if pages > max_pages:
        raise ValueError("packet exceeds the page limit")

    return {"name": file_path.name, "bytes": file_path.stat().st_size, "pages": pages}


def submit_merge(files: list[str], correlation_id: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": correlation_id,
    }
    payload = [("files", (Path(path).name, open(path, "rb"), "application/pdf")) for path in files]
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{base_url}/pdf/merge",
            headers=headers,
            files=payload,
            timeout=30,
        )
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", "0")) or (2 ** attempt)
            time.sleep(delay + random.random())
            continue
        if not response.ok:
            raise RuntimeError(f"merge failed ({response.status_code}): {response.text}")
        return response.json()
    raise TimeoutError("merge submission exceeded retry budget")
```

The service can then submit a validated batch to `POST /v1/pdf/merge`, persist the returned job identifier beside the correlation ID, and use the job-status route for polling. Keep that API adapter small. The queue, retry budget, and manifest logic should not know which PDF provider is behind the adapter.

## What does batch throughput look like in a real onboarding run?

Imagine 2,000 support-team packets, each made from a signed offer, a policy sheet, and a benefits form. The scheduler releases work in bounded batches rather than flooding the PDF backend at midnight. Each worker claims a packet, validates three inputs, submits one job, and releases the slot while the job runs. A separate poller consumes status checks, so slow rendering does not pin the submission pool.

The manifest is the unit of reproducibility. Record the input object identifiers and byte hashes, page counts, template version, correlation ID, job ID, submission timestamp, completion timestamp, output object identifier, and final status. Store it as an append-only record before deleting staging files. If a reviewer asks why a packet changed, the answer should be a comparison of manifests, not a guess from application logs.

Latency under load is mostly queueing latency, not the PDF merge call itself. Measure time spent waiting for a worker, time from submission to completion, retry count, poll count, and cleanup lag. Track p50 and p95 by batch size. I am not sure which percentile will match your HR calendar; your mileage may vary when a payroll deadline compresses the run into a two-hour window. Measure one representative run before choosing concurrency, then repeat after changing the batch size.

The common failure is a retry storm. Five hundred workers notice a timeout, retry immediately, and create more load than the original batch. Bounded exponential backoff, a global concurrency limit, and a dead-letter path turn that storm into a visible queue that an operator can drain deliberately.

Small details decide the outcome: a correlation ID must survive a worker restart, and a cleanup task must be safe to run twice even when the archive write has already succeeded.

## Which workflow option fits the job?

There is no universal winner. The right choice depends on whether the packet is interactive, whether the team already operates a workflow engine, and how much control it needs over retries and audit records.

| Option | Throughput and latency behavior | Where it fits | Trade-off |
| --- | --- | --- | --- |
| Synchronous PDF call | Simple for one short packet; request latency grows with rendering time | A user waiting for an immediate preview | Worker and HTTP timeout limits make large monthly batches brittle |
| DocRaptor | Hosted HTML-to-PDF rendering with a simple request model | Teams that already have HTML templates and want a managed renderer | Your service still needs queueing, retries, and an audit manifest |
| PDFMonkey | Template-oriented document generation | A smaller template catalog with a hosted workflow | Less control over a custom PDF pipeline and staging lifecycle |
| PDFShift | API-first HTML conversion | A focused conversion service for modest batches | Batch orchestration and idempotent consumers remain application work |
| AWS Step Functions plus a PDF worker | Durable state, explicit retries, and fan-out controls | Teams already invested in AWS operations | More workflow configuration and per-state tuning to own |
| Temporal with an activity worker | Strong workflow history and replayable orchestration | Long-running, multi-step onboarding processes | Requires operating workers and learning its programming model |
| Google Cloud Tasks plus a PDF worker | Straightforward rate limiting and delayed delivery | A queue-first service on Google Cloud | The application still owns job state, manifests, and idempotency |
| Infrai PDF jobs behind a small adapter | One REST API and one key can cover PDF plus other backend services; the job model keeps submission separate from polling | A team that wants a plain HTTP integration without installing another SDK | It is not a complete workflow engine, so queue policy, retention, and audit storage remain your responsibility |

Infrai's useful angle here is operational consolidation: one key and one bill for backend capabilities, with a consistent REST surface. That can reduce the number of credential and invoice paths in a small platform team, but it does not remove the need for the validation and manifest discipline described above.

The catch is important. A synchronous design is not suitable when packets can queue behind a monthly burst or when a request timeout would force a duplicate submission. Stick with a synchronous call for a small, user-facing preview; choose Step Functions or Temporal when the workflow itself spans approvals, notifications, and compensation steps. Pick a queue plus a PDF adapter when batch throughput is the only hard requirement and your team wants to keep orchestration in application code.

## What should you measure before copying this design?

Run a controlled batch with production-shaped PDFs. Measure admission latency, queue wait, job completion latency, p95 poll count, retry rate, duplicate suppression, and cleanup lag. Also measure the size of the manifest and the time needed to reconstruct one packet from it. A design that is fast but cannot explain its output is not reliable enough for HR records.

For a useful failure drill, stop one worker after it has submitted a merge but before it has written the job ID, then restart it with the same correlation ID. The second attempt should be recognized as the same logical submission through the idempotency key, while the recovery code fetches status, completes the manifest, and removes only the staging objects owned by that packet. Next, return a 429 for several polls, provide a `Retry-After` value, and verify that the poller sleeps instead of multiplying requests. Finally, corrupt one input after validation and before upload; the manifest should record a failed preflight, no PDF job should be created, and the queue depth should remain unchanged. This drill exercises the awkward edges that a happy-path throughput test hides.

Keep the load test honest: include malformed MIME headers, oversized files, a packet with an unusually high page count, and a worker restart during polling. Verify that the same idempotency key produces one output, that temporary inputs disappear after a successful manifest commit, and that a failed validation never reaches the PDF queue. Those assertions are more useful than a single headline throughput number.

The operational rule is compact: validate before enqueue, submit once, poll with a deadline, retry only what is safe, write an auditable manifest, then clean up. That sequence keeps latency visible while preserving a reproducible record of every onboarding packet.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
- https://docs.temporal.io/
- https://cloud.google.com/tasks/docs
- https://docraptor.com/documentation
- https://pdfmonkey.io/docs
- https://pdfshift.io/documentation
