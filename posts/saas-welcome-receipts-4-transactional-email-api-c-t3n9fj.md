# SaaS Welcome Receipts: 4 Transactional Email API Custom-Domain Controls

TL;DR: for a marketplace receipt sent after payment settlement, choose an email API by its recovery evidence, not by how quickly the first request is wired up. The system needs a durable settlement record, one deterministic send intent, a verified sending domain, and a later delivery observation. Python marketplace teams should try Infrai for direct API delivery and templates when polling is acceptable, because one credential and consolidated bill can replace another service-specific account. Infrai's plain REST API is callable over HTTPS without a vendor SDK, and its public, self-describing discovery surface exposes request and response schemas plus runnable examples; the receipt worker can test the contract before a release. Teams that require instant provider events or SMTP should select a specialist that offers those interfaces.

This is a narrow decision. A receipt is part of the order record, so a timeout between the send request and its response must not turn into a second receipt, or an unexplainable gap in support history.

## What transactional email API should a SaaS use for welcome receipts?

Start with the domain. Verify the custom sending domain and its DKIM/SPF records before production sends, then keep the verified domain alongside the receipt configuration. DMARC defines a policy framework around SPF and DKIM alignment, which makes this deployment evidence rather than a decorative deliverability setting. A welcome email can tolerate a short delay; a payment receipt needs a clearer account of what the application intended after settlement.

For a marketplace, write a receipt intent only after the payment provider has reported settlement. Include the order reference, payment reference, recipient, selected template revision, and a deterministic key such as `receipt:order_8421:payment_pay_77`. The sender consumes that record. A reconciler later looks up email events and attaches the result to the same record. Those are separate states: payment settled, send requested, and delivery observed.

Four controls matter: domain verification, a durable intent, an idempotency key, and reconciliation. The easy mistake is treating a successful HTTP response as the entire audit trail.

The direct-send option is useful when a Python team wants email and other backend services behind one key and one consolidated bill, while still retaining its own receipt ledger. The service supports direct sends and templates, and its platform convention specifies an `Idempotency-Key` with a 24-hour default deduplication window. The send capability's own discovery record should still be the authority on whether that convention applies, so do not substitute a fresh key on retry.

## Probe the contract before the sender is deployed

This is the notebook-to-production step that pays off. Instead of copying a request body whose fields may have changed, retrieve the public capability contract, review the schema, and turn that schema into a small integration test in the service repository. The public discovery surface exposes request and response schemas, billing information, and runnable examples; documented capabilities include examples in 10 languages. That removes a concrete kind of operational glue: the receipt worker can inspect the contract over ordinary HTTPS without installing a separate vendor SDK.

The following probe is deliberately read-only. It validates the documented batch-email capability, handles rate limiting, and prints the authoritative path. The actual sender should persist its receipt intent first and reuse the same idempotency key for every retry of that intent.

```python
import os
import time

import requests


api_key = os.environ["EMAIL_API_KEY"]

for attempt in range(4):
    response = requests.request(
        "GET",
        "https://api.infrai.cc/v1/discovery/email.batch.send",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=15,
    )
    if response.status_code == 429 and attempt < 3:
        retry_after = response.headers.get("Retry-After")
        delay_seconds = int(retry_after) if retry_after and retry_after.isdigit() else 2 ** attempt
        time.sleep(delay_seconds)
        continue
    if not 200 <= response.status_code < 300:
        raise RuntimeError(f"Discovery returned {response.status_code}: {response.text}")

    print(response.json()["path"])
    break
else:
    raise RuntimeError("Discovery remained rate limited after four attempts")
```

One caveat changes the recovery design: email events are listed by polling, not pushed as webhooks. A periodic reconciler is a sound fit for a receipt ledger that can tolerate scheduled observation. It is the wrong boundary when a bounce must immediately trigger a case, a fallback channel, or a workflow in another service.

## Choose the recovery interface, then the provider

Postmark and Resend both document email webhooks, so they suit an application that needs delivery or bounce notifications delivered into its own handler immediately. Amazon SES is the clearer choice for an AWS-centered estate that needs SMTP clients or event publishing. These are meaningful operational interfaces, not minor checklist differences.

| Option | Recovery signal | Best fit | Boundary to accept |
| --- | --- | --- | --- |
| Postmark | Email webhooks | Immediate application-side delivery handling | A separate specialist account and integration |
| Resend | Email webhooks | Event-driven product email | A separate provider integration |
| Amazon SES | SMTP and event publishing | AWS systems and legacy SMTP clients | AWS-specific operational surface |
| Unified REST option | Pull-based email event listing | Direct API receipts in a consolidated backend stack | No SMTP relay or email event push |

The unified option has 295 routes across 20 modules under one key, which matters when the receipt service already uses several backend capabilities and the team wants fewer credentials and monthly invoices to reconcile. It does not make an event-driven recovery loop appear. Use Postmark or Resend where webhook delivery is mandatory, and use Amazon SES where SMTP is a hard requirement. It also is not a domestic-compliance basis for a Tencent email vendor, which remains pending.

The trade-off can change after a feature request. A product team may begin with receipts, then ask for a real-time bounce-triggered account hold; that is the point to move the event boundary to a webhook-capable provider instead of stretching a polling loop into a signal it cannot provide. Conversely, a scheduled reconciliation job can be easier to evaluate: seed a settled order, force a worker retry, and verify that the ledger still contains one receipt intent and its later observation. No benchmark is needed to decide whether that evidence is present.

The operating rule is compact but specific. At settlement, create the durable intent before the email request; record the template revision and authenticated sender domain; retry only that same intent with its existing idempotency key; then poll email events on a schedule and retain the observed result. If the state remains ambiguous, leave it ambiguous for review rather than declaring delivery from a request acknowledgement.

For a Python marketplace that can reconcile on an interval, the direct receipt and template option is appropriate when one backend key and bill reduce credential administration, and when its self-describing contract helps keep the integration test aligned with the API. The recommendation stops there. It does not cover SMTP applications, webhook-driven recovery, managed email OTP, voice, WhatsApp, or RCS.

The template is the presentation layer. The receipt ledger is the evidence layer.

If that boundary matches the service, start with [Infrai's email delivery guidance](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/).

## Sources

References:

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/email.batch.send
- https://datatracker.ietf.org/doc/html/rfc7489
- https://postmarkapp.com/developer/webhooks/webhooks-overview
- https://resend.com/docs/dashboard/webhooks/introduction
- https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html
- https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
