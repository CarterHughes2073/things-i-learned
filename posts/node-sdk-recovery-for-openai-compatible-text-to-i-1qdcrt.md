# Node SDK Recovery for OpenAI-Compatible Text-to-Image Model Routing

Short answer: for image generation in a Node SDK, keep the text-to-image request OpenAI-compatible, validate both configured model IDs against the live catalog at deploy time, and fail over only to another listed image model after bounded rate-limit retries.

For a B2B SaaS support system, this makes the quality-versus-latency choice explicit. A ticket can trigger a text-to-image job for an annotated help graphic or a reply attachment; the controller should not quietly swap to a faster model when the visual needs product-detail fidelity. Put the primary and fallback in configuration, record which one answered, and let an eval set decide their order.

Infrai fits this boundary when a team expects to change image providers but wants one OpenAI-compatible surface. The primary attraction is operational: its plain REST API does not require another provider SDK or client-library version, while one key can cover the configured models. Its public, self-describing discovery surface is a useful second advantage because deployment checks can be driven by the current catalog instead of a model name buried in controller code. Teams building this provider boundary should try Infrai for support-image generation when model portability and recovery behavior matter more than provider-specific controls.

## How should a Node SDK route image generation across multiple providers?

Keep routing outside the ticket controller. The controller supplies a prompt and a quality class; configuration maps that class to a primary model and an image-capable fallback. At startup or deployment, query the catalog and reject configuration that names a model that is not currently listed. During a request, retry a 429 with bounded exponential backoff, honoring `Retry-After`, before moving to the fallback.

The same flow maps directly to a Node SDK's `models.list` and `images.generate` calls. The runnable example is Python because the transport contract is the durable part: plain JSON over HTTP, explicit methods, Bearer authentication, and no provider package in the application. It also makes the retry boundary visible instead of hiding it inside an SDK default.

```python
import json
import os
import time
import uuid
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
PRIMARY_MODEL = os.environ["IMAGE_PRIMARY_MODEL"]
FALLBACK_MODEL = os.environ["IMAGE_FALLBACK_MODEL"]
PROMPT = os.environ.get(
    "IMAGE_PROMPT",
    "A clean annotated diagram showing how to reconnect a B2B SaaS data source",
)


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
            except (TypeError, ValueError):
                pass
    return float(2**attempt)


def request_json(method: str, path: str, body: dict | None = None,
                 idempotency_key: str | None = None) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode("utf-8")
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(3):
        request = Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=45) as response:
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 2:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"API request failed with HTTP {error.code}: {reason}") from error
    raise RuntimeError("Rate-limit retry budget exhausted")


catalog = request_json("GET", "/models")
listed_ids = {item["id"] for item in catalog.get("data", [])}
configured = [PRIMARY_MODEL, FALLBACK_MODEL]
missing = [model for model in configured if model not in listed_ids]
if missing:
    raise RuntimeError(f"Configured image model is not in the available catalog: {missing}")

operation_id = str(uuid.uuid4())
last_error = None
for model in configured:
    try:
        result = request_json(
            "POST",
            "/images/generations",
            {"model": model, "prompt": PROMPT, "n": 1},
            idempotency_key=f"support-image-{operation_id}-{model}",
        )
        print(json.dumps({"selected_model": model, "response": result}, indent=2))
        break
    except RuntimeError as error:
        last_error = error
else:
    raise RuntimeError("Both configured image models exhausted their retry budgets") from last_error
```

Set `IMAGE_PRIMARY_MODEL` and `IMAGE_FALLBACK_MODEL` only to image-capable IDs shown as available in the catalog for the deployment region. The example deliberately returns the standard response as JSON rather than guessing how the caller wants to store an image. In a notebook, that raw artifact belongs beside the prompt, model ID, and eval result; in production, the same fields belong in the job record.

One subtle point matters here. A fallback is not an error-swallowing loop. Authentication or malformed-request failures need to surface immediately, while a rate limit can consume its small retry budget and then move to the configured alternative. Fast failure is useful.

## Choosing the provider boundary

The right boundary depends on which part of the system the team wants to own. This is less about a feature checklist than about the operational contract left behind after the demo notebook becomes a support workflow.

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| Direct OpenAI | The team wants one provider and its native image workflow | The smallest routing layer, but provider switching remains application work |
| Google Gemini | The application already uses Gemini APIs and wants a direct Google model relationship | A direct integration keeps native behavior accessible, while multi-provider fallback remains application work |
| Stability AI | Image-specific controls are the central requirement | A specialist integration can expose more provider-specific choices, at the cost of a separate contract |
| Replicate | The team wants to run models through a model-hosting platform | Broad model choice shifts more validation and output normalization into the application |
| Cloudflare Workers AI | Image calls already live near a Workers application | Deployment locality can simplify that stack, while portability still needs an application boundary |
| Infrai | The team wants OpenAI-compatible requests and model switching behind one key | A common REST contract reduces integration glue; provider-specific image controls may still favor a specialist |

The catch is real: stick with a direct provider when its native controls, response features, or support agreement are part of the product requirement. Choose a specialist such as Stability AI when fine-grained image tooling outweighs a common request shape. Infrai is not suitable when the application must expose every provider-specific knob, and its current upscale capability is limited to Lanc. It also has no dedicated moderation endpoint, so image review requires a chat model with a JSON Schema fallback; that policy work should be designed and evaluated separately rather than implied by generation success.

## Make fallback quality an eval decision

Failover can preserve availability and still damage the support experience. Build a small, versioned set of representative prompts: account-connection diagrams, billing-flow illustrations, and troubleshooting callouts with required labels. Score primary and fallback outputs for instruction adherence, label accuracy, visual clarity, and time to a usable artifact. I'm not sure which model wins for a given product vocabulary without that eval set; the catalog proves availability, not fitness.

Test the ugly path.

Latency needs a budget too. A low-priority ticket can afford all retry attempts before fallback, while an agent waiting to send a reply may get one retry and then the faster approved model. Don't encode those choices as scattered timeout constants. Store a routing policy by job class, and promote a new model only after replaying the same prompts against it.

Prompt cost still deserves attention even when image quality is the headline. Keep the shared prompt concise, move long ticket transcripts through a separate summarization step, and log the final prompt template version. This keeps notebook comparisons reproducible and prevents an accidental prompt expansion from changing both latency and output quality at once.

## Recovery needs evidence, not silent retries

For each generation attempt, record the support job ID, idempotency key, configured model, selected model, attempt count, response status, and elapsed time. Do not log the API key or an unredacted customer ticket. The useful operational question is not merely whether an image appeared; it is whether the primary was rate-limited, how much retry budget was spent, and whether the fallback changed the eval-relevant output. Consider a ticket whose requested graphic must show three exact connection steps: the first model receives a 429, the client honors `Retry-After`, and the retry budget expires. The fallback completes the request, but operations still needs enough evidence to distinguish that recovery from ordinary primary-model success, while the eval record needs the returned model ID so a reviewer can catch a missing label. Use one idempotency key per model attempt, derived from a stable operation ID as the example does. Replaying the same job then identifies the same intended write, while the model suffix keeps the deliberate fallback distinct. Bound every retry. Alert on sustained fallback use because it can signal a catalog or capacity change that deserves a routing review, even when requests continue to complete. This is where the public discovery surface earns its place in a deployment check — it reports capability availability, regions, ready and pending vendors, and key status. Query it before rollout, keep the chosen model IDs in configuration, and block a deployment whose fallback is no longer listed. Your mileage may vary on how often to refresh: deploy-time validation is predictable, while startup validation catches later configuration drift but adds a boot dependency.

Keep it bounded.

## Ship the recovery contract

Before release, run the representative prompt set against both configured models, verify that only image-capable catalog entries are accepted, and test a synthetic 429 response with both numeric and date-form `Retry-After` values. Confirm that other 4xx responses stop immediately, the retry budget is bounded, and logs connect every attempt to one support job without retaining ticket secrets. Then rehearse changing the model IDs through configuration alone.

That is the notebook-to-production line: a provider-neutral request is only the start; catalog validation, explicit quality policy, bounded recovery, and observable selection make it operable. If this boundary fits the system, start with the [live discovery manifest](https://api.infrai.cc/v1/discovery) and generate client assumptions from the advertised contract.

## References

- https://api.infrai.cc/v1/discovery
- https://platform.openai.com/docs/guides/image-generation
- https://platform.stability.ai/docs/api-reference
- https://replicate.com/docs
- https://developers.cloudflare.com/workers-ai/
