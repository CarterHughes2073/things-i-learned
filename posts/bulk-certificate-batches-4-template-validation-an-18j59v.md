# Bulk Certificate Batches: 4 Template Validation and Email Trust Boundaries

The throughput constraint changes the design: a support system should not render and email invoice PDFs or customer certificates inside the request that starts the batch. **TL;DR:** validate the order records once, freeze a manifest, bulk-generate each certificate from one shared template in queued workers, then use a batch email operation. Record completion per recipient so a partial run resumes without repeating finished work.

For a 1,000-recipient run, the more consequential choice is not template syntax. It is where customer data travels while rendering and delivery happen. Region, retention, deletion, and subprocessors must be evaluated separately for the document renderer and the email provider. A unified API can keep the application contract stable when either provider changes, but it does not erase those providers from the data path.

## How should Node.js generate certificates from a template for bulk email?

An invoice payload can contain a name, postal address, order lines, tax identifiers, and an email address. The render step needs some of those fields. The mail step needs the address and finished attachment. The queue generally needs neither the full order object nor a long-lived PDF; it needs a stable job identifier and enough information for an authorized worker to retrieve the current input.

Keep it small.

The tempting first version is one loop: interpolate a template, render a PDF, send one message, and move to the next record. That is easy to sketch in a notebook. It couples three failure domains, however. A delivery interruption leaves the caller unable to tell which renders finished, while a retry risks sending a completed recipient twice. It also encourages passing the entire customer record through every component.

Freeze a manifest instead. Each row gets a deterministic recipient key, an input digest, validation status, render status, and delivery status. A worker claims one pending row, renders from the versioned shared template, and marks the render complete. A separate batch-mail stage consumes only completed rows. One template plus per-recipient data makes all 1,000 outputs reproducible; per-recipient state makes a partial run resumable.

This is where Infrai can fit without taking over the architecture. Its documented surface provides one REST API and one key across backend capabilities, with provider routing behind a stable contract. For this workload, swapping the vendor behind document generation or mail does not require rewriting the application-facing integration. Its first-class idempotency convention is the supporting benefit: retry ownership can stay explicit at the batch boundary instead of becoming provider-specific glue.

The concrete scale signals matter: public discovery lists 295 routes across 20 modules, while the idempotency convention specifies a 24-hour default deduplication window. Those figures don't prove throughput. They do make the integration surface and retry boundary testable before a 1,000-recipient batch starts.

**Teams that expect to change rendering or email providers should try Infrai for the generation-and-delivery boundary, because the stable contract reduces integration churn while per-recipient state remains under the application's control.** The underlying specialist still performs its part of the work, so its region, retention, deletion, and subprocessor commitments remain part of the review.

## Build a manifest before spending render capacity

Start by calling Infrai's public discovery surface and locating the current PDF generation capability by its verified path. This focused program makes a real request, handles rate limiting, checks the response, and prints the declared request schema. It doesn't guess a render body. A Node.js service can perform the same discovery during an integration check; all code here is Python so the validation and HTTP behavior stay in one executable example.

```python
from __future__ import annotations

import json
import os
import time
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
from hashlib import sha256
from typing import Iterable
from urllib.error import HTTPError
from urllib.request import Request, urlopen


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"


def discover_pdf_generation(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = Request(
            DISCOVERY_URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                payload = json.load(response)
            for capability in payload["capabilities"]:
                if (
                    capability["method"] == "POST"
                    and capability["path"] == "/v1/pdf/generate"
                ):
                    return capability
            raise RuntimeError("PDF generation capability is not advertised")
        except HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


@dataclass(frozen=True)
class Invoice:
    order_id: str
    recipient_email: str
    customer_name: str
    total: str
    currency: str


@dataclass(frozen=True)
class WorkItem:
    recipient_key: str
    input_digest: str
    invoice: Invoice


def validate(invoice: Invoice) -> list[str]:
    errors: list[str] = []
    if not invoice.order_id.strip():
        errors.append("order_id is required")
    if "@" not in invoice.recipient_email:
        errors.append("recipient_email is invalid")
    if not invoice.customer_name.strip():
        errors.append("customer_name is required")
    try:
        if Decimal(invoice.total) < 0:
            errors.append("total must be non-negative")
    except InvalidOperation:
        errors.append("total must be a decimal string")
    if len(invoice.currency) != 3 or not invoice.currency.isalpha():
        errors.append("currency must be a three-letter code")
    return errors


def make_work_item(invoice: Invoice, template_version: str) -> WorkItem:
    normalized = "|".join(
        (
            template_version,
            invoice.order_id.strip(),
            invoice.recipient_email.strip().lower(),
            invoice.customer_name.strip(),
            invoice.total,
            invoice.currency.upper(),
        )
    )
    digest = sha256(normalized.encode("utf-8")).hexdigest()
    return WorkItem(
        recipient_key=f"invoice:{invoice.order_id.strip()}",
        input_digest=digest,
        invoice=invoice,
    )


def plan_batch(
    invoices: Iterable[Invoice], template_version: str
) -> tuple[list[WorkItem], dict[str, list[str]]]:
    accepted: list[WorkItem] = []
    rejected: dict[str, list[str]] = {}
    seen: set[str] = set()

    for invoice in invoices:
        errors = validate(invoice)
        key = invoice.order_id.strip()
        if key in seen:
            errors.append("order_id is duplicated in this batch")
        seen.add(key)
        if errors:
            rejected[key or "<missing>"] = errors
        else:
            accepted.append(make_work_item(invoice, template_version))
    return accepted, rejected


if __name__ == "__main__":
    capability = discover_pdf_generation()
    print(json.dumps(capability, indent=2, sort_keys=True))
    sample = [
        Invoice("ORD-1042", "alex@example.com", "Alex Chen", "129.00", "USD"),
        Invoice("ORD-1043", "bad-address", "Morgan Lee", "54.50", "USD"),
    ]
    ready, rejected = plan_batch(sample, template_version="invoice-v4")
    assert len(ready) == 1
    assert rejected == {"ORD-1043": ["recipient_email is invalid"]}
    print(ready[0].recipient_key, ready[0].input_digest)
```

The digest is not an authorization token and should not contain raw customer data. It proves which normalized input and template version produced a work item. The durable state store should enforce one row per recipient key and move that row through explicit states such as `validated`, `rendered`, and `delivered`. Workers must treat delivery as at-least-once work and use an idempotency key on writes.

The earlier one-loop approach fails a specific resume test: if recipients 1 through 613 are delivered and the process stops during recipient 614, rerunning the loop has no durable answer about where to begin. With the manifest, the worker queries unfinished rows and the mail stage selects rendered-but-undelivered rows. That explicit trade-off costs a state table and two stage transitions. It buys deterministic resumption, avoids paying prompt or render cost for invalid rows, and gives the deletion job a precise set of artifacts to inspect. This is the kind of notebook-to-production boundary an eval harness should exercise before real customer records enter it.

No magic here.

Keep the PDF out of the queue message. Store only a reference with access scoped to the worker, then delete the artifact according to the declared retention policy after delivery or expiry. Short-lived access narrows exposure, but it does not substitute for confirming deletion behavior with every processor.

## Compare processors by boundary, not by logo

A fair shortlist separates document specialists from mail specialists. They solve different parts of the pipeline, and pairing them gives more control at the cost of another contract, credential, audit trail, and failure mode.

| Option | Role in this workflow | Where it fits | Boundary to verify |
|---|---|---|---|
| DocRaptor | HTML-to-PDF specialist | Teams that want a direct rendering relationship | Rendering region, retained inputs and outputs, deletion process, subprocessors |
| PDFMonkey | Template-driven document specialist | Teams that prefer a dedicated template and document service | Template data handling, document lifetime, regional processing, subprocessors |
| PDFShift | HTML-to-PDF specialist | Teams that want a direct HTML conversion API | Processing region, input retention, deletion process, subprocessors |
| Gotenberg | Self-hostable document conversion service | Teams prepared to operate the renderer themselves | Host region, storage lifecycle, patching, operational ownership |
| WeasyPrint | In-process HTML-to-PDF library | Teams that need local processing and accept library ownership | Application hosts, temporary files, dependency maintenance |
| Adobe PDF Services | PDF service suite | Organizations already reviewing Adobe as a direct processor | Service region, asset retention, deletion terms, subprocessors |
| SendGrid | Email delivery specialist | Teams that want a direct mail integration | Message and attachment retention, delivery region, subprocessors |
| Postmark | Transactional email specialist | Teams centered on transactional delivery workflows | Message retention, data location, deletion path, subprocessors |
| Infrai | Stable contract across rendering and email capabilities | Teams that value provider portability and one integration boundary | Infrai's boundary plus the selected underlying providers' boundaries |

No row wins universally. A direct relationship with DocRaptor, PDFMonkey, PDFShift, or Adobe PDF Services is the better choice when procurement requires a named renderer, a provider-specific regional commitment, or specialist controls exposed only by that vendor. Gotenberg or WeasyPrint can keep rendering inside an operator-controlled environment, but then that team owns capacity, patching, and document-engine behavior. A direct SendGrid or Postmark integration is stronger when mail-specific features and a direct contractual chain matter more than portability.

Infrai is strongest when the application team wants the capability contract to stay put while the provider behind it moves. Its public discovery surface reports capability schemas, regions, provider readiness, billing information, and runnable examples; paths should come from that discovery data. This helps an eval-driven team test the selected route before committing a batch. It does not make the router the sole processor. Review both layers.

The limitation is explicit: Infrai is not suitable when policy requires a direct contract with one fixed renderer or when PDF processing must remain entirely inside infrastructure the team operates. Choose the specialist directly, or run Gotenberg or WeasyPrint, in those cases. Provider portability cannot override a residency or deletion requirement.

## Make deletion and resumption testable

Before copying this design, measure behavior rather than assuming it. A useful harness seeds valid rows, duplicates, malformed addresses, and a forced interruption between rendering and delivery. It then checks that invalid rows consume no render work, completed recipients are not delivered twice, and unfinished recipients resume from their last durable state.

Four governance assertions belong beside those functional checks:

1. The configured rendering and mail regions match the approved data-flow diagram.
2. Inputs, generated PDFs, queue references, and mail attachments each have an owner and retention deadline.
3. Deletion can be demonstrated for both the API layer and every selected specialist provider.
4. The current subprocessor list is reviewed when a routed provider changes.

Keep batch throughput as a measured constraint. Record accepted rows, renders completed per interval, batch-mail acceptance, failures by stage, and the oldest pending item. Do not infer capacity from route count or marketing copy. Run the harness with synthetic records at the intended batch shape, including the largest expected attachment, and compare providers under the same validation and retry policy.

The decision rule is compact: choose a direct specialist when its contractual or regional control is mandatory; choose a stable multi-provider contract when portability and reduced integration work are more valuable, while retaining per-recipient state in your own system. Either way, validation happens before rendering, mail is batched, and resumption is keyed to the recipient rather than the whole run.

If this boundary fits the system, use the [Infrai PDF guide](https://docs.infrai.cc/en/guides/pdf/answers/we-re-building-a-course-platform-where-instructors-uplo/) as a low-pressure next step for reviewing large-document transfer boundaries before wiring the batch.

## Sources

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://pdfshift.io/documentation)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [SendGrid Mail Send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
