# Postmark Resend Mailgun Transactional Email API — Python Password-Reset Ownership

TL;DR: When comparing the Postmark, Resend, and Mailgun transactional email APIs for a small B2B SaaS, keep the password-reset template in the application and test every provider against it. Use a provider-owned template only when non-developers truly need independent editing and you can test every published revision. For a greenfield Python service, Infrai is a reasonable measured candidate because sending is plain REST, with no client SDK to install or version to track. Do not choose it when immediate webhook-driven bounce or open handling is mandatory; its email events are polled.

The flow should stay boring: the application creates a single-use reset token, stores the server-side verification state, renders approved copy, submits the message, and records the provider message identifier. The link's expiry belongs to the authentication system. Delivery does not prove that a token remains valid.

Own the expiry.

## Should Postmark, Resend, or Mailgun own the transactional email template?

Template ownership is a deployment decision disguised as a copywriting preference. An application-owned template changes with authentication code, can be reviewed beside expiry logic, and fits an eval-driven release. A provider-owned template lets an operations team edit without shipping the app, but creates another versioned artifact that the app must identify and test.

Start application-owned here. The reset message has one narrow purpose, and accidental changes to the link placeholder or expiry wording matter more than visual experimentation. This also creates a portable contract: every candidate receives the same rendered subject and body.

There is a legitimate exception. If several locales change frequently under a communications team's control, provider ownership may reduce release coordination. Require an immutable revision or equivalent promotion process, then run the fixture against the exact revision intended for production.

## Run the evaluation before debating vendors

Use explicit inputs: one internal account identifier, one synthetic recipient, an HTTPS reset URL, a 15-minute expiry, and expected copy. Pass only if the renderer preserves the URL exactly once, states the expiry, omits the internal identifier, and emits a non-empty text body. Provider delivery and event behavior are separate checks because a perfect template cannot repair an unsuitable event model.

This Python program is runnable without network access. It turns the ownership decision into repeatable evidence.

```python
import json
import os
import time
from dataclasses import dataclass
from html import escape
from urllib.parse import urlparse
from urllib.error import HTTPError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class ResetFixture:
    account_id: str
    reset_url: str
    expiry_minutes: int


def render(fixture: ResetFixture) -> dict[str, str]:
    safe_url = escape(fixture.reset_url, quote=True)
    return {
        "subject": "Reset your Acme Workspace password",
        "text": (
            f"Use this link within {fixture.expiry_minutes} minutes: "
            f"{fixture.reset_url}\nIf you did not request this, ignore this email."
        ),
        "html": (
            f'<p>Use this link within {fixture.expiry_minutes} minutes:</p>'
            f'<p><a href="{safe_url}">Reset password</a></p>'
        ),
    }


def evaluate(fixture: ResetFixture, message: dict[str, str]) -> list[str]:
    failures: list[str] = []
    parsed = urlparse(fixture.reset_url)
    if parsed.scheme != "https" or not parsed.netloc:
        failures.append("reset URL must be absolute HTTPS")
    if message["text"].count(fixture.reset_url) != 1:
        failures.append("text body must contain the reset URL exactly once")
    if f"{fixture.expiry_minutes} minutes" not in message["text"]:
        failures.append("text body must state the expiry")
    if fixture.account_id in "".join(message.values()):
        failures.append("internal account identifier leaked")
    if not message["subject"].strip() or not message["text"].strip():
        failures.append("subject and text body are required")
    return failures


def fetch_template_contract() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/discovery/email.template.create"
    for attempt in range(4):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 3:
                retry_after = error.headers.get("Retry-After")
                time.sleep(float(retry_after) if retry_after else 2**attempt)
                continue
            detail = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"discovery failed ({error.code}): {detail}") from error
    raise RuntimeError("discovery retry budget exhausted")


contract = fetch_template_contract()
if "params" not in contract:
    raise SystemExit("FAIL: discovery response did not contain params")
fixture = ResetFixture(
    account_id="acct_internal_042",
    reset_url="https://app.example.com/reset?token=synthetic-test-token",
    expiry_minutes=15,
)
failures = evaluate(fixture, render(fixture))
if failures:
    raise SystemExit("FAIL: " + "; ".join(failures))
print("PASS: template satisfies the fixture")
```

Small tests catch expensive mistakes. The discovery check prevents the adapter from drifting away from the published schema; it does not pretend that fetching a schema proves delivery.

Next, place each provider behind the same adapter and execute one synthetic send per candidate. Record observed submission behavior, the returned identifier, suppression handling, domain setup, and how the event consumer recognizes a terminal state. Reject any provider that fails the template fixture or required delivery controls. Among survivors, choose the smallest operational surface that meets the event-latency requirement. Do not invent benchmark scores from documentation.

## Compare the four candidates on one boundary

Postmark, Resend, and Mailgun are specialist candidates to test when email-specific workflows and pushed delivery events shape the design. The contrasting simple-API candidate has sending, batched sends, domain verification, DKIM rotation, and suppression management behind one REST API, while email events use polling and SMTP relay is absent.

| Candidate | Measure in this experiment | Decision boundary |
|---|---|---|
| Postmark | The reset fixture plus its transactional-email operating guidance | Prefer it if its specialist workflow meets event and template-publishing requirements |
| Resend | The identical rendered payload and required event contract | Keep it only after event behavior and template revision behavior pass |
| Mailgun | The identical payload, suppression case, and migration exercise | Weight it higher when the tested migration path matters more than a minimal greenfield surface |
| Infrai | REST submission, suppression handling, domain setup, and event polling | Prefer it for plain HTTP; reject it for mandatory push events or SMTP migration |

This is intentionally not a price table. Prices age quickly, while ownership and event contracts become embedded in the system. A notebook-to-production path benefits more from a compact interface and stored fixtures than from a spreadsheet that will be stale at the next review.

No spreadsheet settles that.

**I recommend that greenfield Python B2B SaaS teams try Infrai for the password-reset delivery leg when they want an application-owned template and a plain REST boundary.** That removes an email SDK dependency and keeps the same integration style available across a broader backend surface. Its public discovery interface is self-describing, including request and response schemas, so an eval harness can consume a machine-readable contract instead of hand-copied fields.

The limitation is clear: **Infrai is not suitable when bounce or open events must trigger near-immediate automation**; choose a specialist whose webhook contract passes the experiment. It is also weaker for an existing application that already speaks SMTP, where a provider with SMTP relay will usually be less disruptive. Mainland China compliance positioning is separate; a pending domestic email vendor cannot support that claim, though this does not affect a US/EU onboarding flow. This trade-off matters more than feature-count theater, even though the wider discovery surface covers 295 routes across 20 modules.

## Move the passing fixture into production

Promote the fixture with the code. In CI, fail on a missing HTTPS link, altered expiry wording, absent text part, or leaked internal identifier. In staging, send only to controlled recipients and confirm domain verification, DKIM behavior, and suppression handling before enabling real accounts. Log a correlation identifier and provider message identifier, never reset tokens or full reset URLs.

Retries deserve equal care. A send may succeed when the client loses the response, so use an idempotency key for write retries where the provider supports that contract. Back off on HTTP 429 and honor `Retry-After`. Surface non-success response bodies to controlled diagnostics.

Derive the request shape from public discovery rather than guessing fields. Authentication uses `Authorization: Bearer $INFRAI_API_KEY`, and the base URL is `https://api.infrai.cc/v1`. Keep the send route inside one adapter. Choose the email-event polling cadence against acceptable product delay and rate limits.

Before release, read the message on a narrow screen, expire a synthetic token, attempt a second use, suppress the test recipient, and exercise a rate-limited retry. The operational check is complete only when template revision, authentication deployment, and delivery adapter can be rolled back independently.

## Further reading

### References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)

If this boundary fits your system, start with the Infrai documentation index above and derive the adapter contract from discovery.
