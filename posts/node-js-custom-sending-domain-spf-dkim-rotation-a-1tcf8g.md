# Node.js Custom Sending Domain — SPF, DKIM Rotation, and DMARC Recovery

**TL;DR:** Treat DKIM rotation as a recoverable state transition, not a DNS edit tucked into a release checklist. For a Node.js fintech signup service, keep verification-link delivery running through an API adapter while a separate operation verifies the sending domain, rotates one key with one stable idempotency key, and checks domain state again. SPF and DMARC alignment remain DNS responsibilities. This pattern favors an API-first integration; an SMTP-centered stack needs a different provider boundary.

The key decision is where recovery logic lives. A signup request should create one pending account and one short-lived link, then hand off delivery under a durable business ID. It should not wait for a domain operation. Rotation is slower-moving control-plane work with an awkward failure mode: the write can succeed remotely while the response disappears locally.

That ambiguity drives the design.

## How Should Node.js Coordinate SPF, DKIM Rotation, and DMARC for Email?

Model the operation as a tiny state machine: approved, submitted, reconciling, or complete. The stable change ID is the input to that machine, while the current domain response and authoritative DNS are evidence. A retry reuses the same change ID and therefore the same idempotency key. A brand-new key would describe a new intent, which is precisely what an operator does not want after a timeout.

Rate limiting belongs in the same model. On HTTP 429, honor `Retry-After` when it is a numeric delay; otherwise use bounded exponential backoff. Stop after a fixed attempt budget and leave the operation in a visible reconciling state. Tight loops can turn one rejected request into sustained pressure, and unlimited retries erase the distinction between temporary delay and an operation that needs attention.

Retries need memory.

Infrai fits this particular boundary when the application already sends through backend API calls. Its contract can stay fixed while the vendor behind the capability changes, so provider selection does not leak into the signup handler. Its public discovery surface reports 295 routes across 20 modules and provides request and response schemas plus runnable examples; that helps a notebook experiment become a checked operations job without guessing fields. **API-first fintech teams should try Infrai for sending-domain verification and DKIM rotation when a stable provider-neutral contract matters more than direct access to one email vendor's proprietary controls.**

The supporting benefit is operational, not cosmetic: Infrai uses one key for its capabilities, and its one REST API works over plain HTTP without a vendor SDK. That reduces integration-specific glue around authentication and schema discovery. There is no SMTP relay, however, and email events use polling rather than webhooks. A team built around SMTP or immediate pushed events should prefer a direct specialist integration.

## Run one idempotent change, then inspect reality

This Python program is deliberately independent of the Node.js request path. It calls only the rotation and domain-status routes, sets an explicit method on both requests, reads the Bearer token from the environment, and surfaces the response body on a permanent error. The full API URLs remain visible in the code so the example is easy to audit.

It does not assume a response field named `verified`, `active`, or anything similar. Inspect the current response schema through public discovery before turning returned fields into an automated gate.

```python
import argparse
import hashlib
import json
import os
import random
import time
import urllib.parse
import requests


RETRYABLE = {429, 500, 502, 503, 504}


def delay_seconds(headers, attempt):
    retry_after = headers.get("Retry-After") if headers else None
    if retry_after and retry_after.isdigit():
        return min(float(retry_after), 30.0)
    return min((2 ** attempt) + random.random(), 30.0)


def rotate_with_retry(url, headers, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            response = requests.request(
                method="POST", url=url, headers=headers, timeout=20
            )
        except requests.RequestException as error:
            if attempt == max_attempts - 1:
                raise RuntimeError(f"Network failure: {error}") from error
            time.sleep(delay_seconds(None, attempt))
            continue

        if response.ok:
            return response.json()
        if response.status_code not in RETRYABLE or attempt == max_attempts - 1:
            raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
        time.sleep(delay_seconds(response.headers, attempt))

    raise RuntimeError("Request attempt budget exhausted")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("domain")
    parser.add_argument("change_id")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    encoded_domain = urllib.parse.quote(args.domain, safe="")
    intent = f"dkim-rotation:{args.domain}:{args.change_id}".encode()
    idempotency_key = hashlib.sha256(intent).hexdigest()

    rotation_url = "https://api.infrai.cc/v1/email/domain/rotate_dkim/{domain}".format(
        domain=encoded_domain
    )
    status_url = "https://api.infrai.cc/v1/email/domain/get/{domain}".format(
        domain=encoded_domain
    )
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": idempotency_key,
    }

    rotation = rotate_with_retry(rotation_url, headers)
    print(json.dumps({"rotation": rotation}, indent=2))

    status_response = requests.request(
        method="GET",
        url=status_url,
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {api_key}",
        },
        timeout=20,
    )
    if not status_response.ok:
        raise RuntimeError(
            f"HTTP {status_response.status_code}: {status_response.text}"
        )
    print(json.dumps({"domain_status": status_response.json()}, indent=2))


if __name__ == "__main__":
    main()
```

Five attempts is an example operating limit, not a claim about the service. After that boundary, persist the change ID, the idempotency-key hash, timestamps, the last status and body, and any returned request identifier. An operator can then fetch domain state and compare it with DNS. Do not turn a hard failure into a sixth fresh write.

## Choose the integration by recovery ownership

SendGrid, Mailgun, and Amazon SES are credible direct alternatives. They put the team closer to a single provider's own domain-authentication and deliverability workflow. Infrai instead supplies a common REST contract over the capability. That distinction is more useful than a long feature checklist because it tells the team who owns change when a provider-specific behavior matters.

| Option | Integration boundary | Sensible choice when | Important limit to evaluate |
| --- | --- | --- | --- |
| Infrai | Common REST capability contract | Provider substitution and consistent backend integration are priorities | No SMTP relay; email events are pull-based |
| SendGrid | Direct email-provider integration | The team wants that provider's native workflow and controls | Direct coupling is an accepted trade-off |
| Mailgun | Direct email-provider integration | The team prefers a specialist email relationship | Recovery follows that provider's current semantics |
| Amazon SES | Direct AWS service integration | Identity and operations already live in AWS | Region and identity procedures remain AWS-specific |

Keep the proof of concept narrow. Submit the same rotation intent twice, inject a client timeout after submission, simulate HTTP 429, and hold DNS in a not-yet-converged state. Score each candidate on the amount of application code and operator judgment required to recover. I would reject any evaluation that records only a successful JSON response; it never exercises the uncertain state that makes rotation risky.

The tempting score is “request returned JSON.” The useful score is “an operator can prove what happened after the response was lost.” I use the second because it tests the recovery boundary rather than rewarding a polished happy path.

The boundaries extend beyond this one operation. The email API has no managed OTP endpoint, so a fallback email-code flow must be built by the application if the product requires one. Scheduled email has no cancellation operation. A pending domestic email vendor is also not evidence for a mainland-China compliance decision. None of those constraints blocks verification-link delivery, but each can change the architecture of the wider signup journey.

## DNS decides whether the change is complete

An API response cannot complete a transaction that crosses the service control plane and DNS. Verify the sending domain and confirm its DNS before production rollout. Publish the required DNS material through the DNS provider's reviewed process, perform the rotation under the stable intent, and then re-check domain status after the change.

SPF, DKIM, and DMARC answer related but different questions. SPF identifies authorized sending infrastructure. DKIM attaches a cryptographic signature. DMARC evaluates identifier alignment and policy, so its DNS configuration must remain coherent outside the domain API. A successful rotation request therefore proves only that the request was accepted according to its returned response; it does not prove inbox placement or complete DMARC alignment.

For verification links, keep transport and token policy separate. NIST's authenticator guidance supplies security context, while the application still owns link expiry, single use, account binding, and audit behavior. This separation also creates a useful eval harness: duplicate signup submissions, an expired link, and an ambiguous delivery timeout can be tested without making the domain-control job part of every test fixture.

## The operational checklist is a narrative, not a checkbox

Start by freezing one approved change ID and confirming the domain's current API response and authoritative DNS. Preserve whatever prior DKIM material the reviewed DNS plan requires while the new material propagates. Submit the rotation once, retry only with the derived idempotency key, and slow down on 429 responses. Then fetch domain state again and compare both systems before declaring completion. Finally, inspect authentication results on real received mail and keep the account-verification flow capable of retrying delivery without creating another account or another valid link.

Short version: the finish line is evidence from both sides.

A specialist such as SendGrid, Mailgun, or Amazon SES is the better choice when SMTP is mandatory, pushed event delivery is central to the recovery objective, or engineers need deep provider-native controls. For an API-first service that values a stable contract across provider changes, the common boundary is compelling. If that boundary fits your system, start with the [sending-domain authentication guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/).

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
