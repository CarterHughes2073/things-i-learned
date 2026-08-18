# Prompt Safety Before Pixels: Image Generation API Moderation with JSON Schema

For a property-management support workflow, the best image generation API is the one that makes prompt safety and structured output testable, even when the image service has no separate moderation endpoint. Short answer: put a chat model in front of the image API, require a small JSON schema from that model, and treat the image request as a side effect that only runs after validation. This is more dependable than trying to repair an unsafe prompt after an image has already been generated.

## Start with the failure boundary, not the image

An incoming ticket might say that a resident's kitchen has water damage and ask for a visual explanation of the repair process. The system should first classify the request, remove private details, and decide whether an image is appropriate. Only then should it turn the approved intent into an image prompt.

The useful data flow is short: ticket in, structured safety decision, normalized prompt, image request, and an audit record containing the decision and request identifiers. The image itself is a downstream artifact. It must not be the place where policy is decided.

That separation matters because "safe" is not a property you can infer from a successful HTTP response. A service can return a perfectly valid image for an unsuitable request. Your eval harness should therefore score the JSON decision before it scores image quality. This also gives the support team a place to explain a denial without pretending that pixels can carry policy reasoning.

Stop there.

## How should an image generation API handle prompt safety, moderation, and JSON schema?

Use a chat model for the narrow decision step and make its output boring. For example, the application can accept only `allow`, `revise`, or `deny`, plus a sanitized prompt when one is needed. A schema gives the caller a stable contract; it does not magically make the model's judgment correct.

Here is a provider-neutral Python sketch. The `chat_model` and `image_api` objects are adapters owned by the application, so changing providers does not change the policy code.

```python
from dataclasses import dataclass
from typing import Literal


@dataclass
class SafetyDecision:
    action: Literal["allow", "revise", "deny"]
    prompt: str | None
    reason: str


def build_image(ticket_text: str, chat_model, image_api):
    decision = chat_model.parse_json(
        system=(
            "Classify an image request for a property-management support ticket. "
            "Return only the declared JSON schema. Remove personal data. "
            "Deny requests for harmful or exploitative imagery."
        ),
        user=ticket_text,
        schema={
            "type": "object",
            "additionalProperties": False,
            "required": ["action", "prompt", "reason"],
            "properties": {
                "action": {"enum": ["allow", "revise", "deny"]},
                "prompt": {"type": ["string", "null"]},
                "reason": {"type": "string"},
            },
        },
    )

    if decision["action"] == "deny":
        return {"status": "denied", "reason": decision["reason"]}
    if not decision["prompt"]:
        return {"status": "denied", "reason": "Missing sanitized prompt"}

    image = image_api.generate(prompt=decision["prompt"])
    return {
        "status": "generated",
        "image_id": image["id"],
        "policy_action": decision["action"],
    }
```

The adapter contract is the important part. `parse_json` must reject malformed output rather than silently coercing it, and `generate` must return a stable image identifier that can be written to the ticket record. If an API offers no moderation endpoint, this preflight is a reasonable architecture, but it is still a policy layer you own: test it, log it, and provide a human review path for ambiguous cases. A schema is a guardrail around communication between components, not evidence that the classification itself is right.

## What breaks when the safety contract is vague?

The first failure is usually field drift. A model returns `safe: true` in one test and `action: "allow"` in another. The application then treats a missing field as permission to continue. Reject missing fields and unexpected fields at the boundary; a failed decision should stop generation, not become an implicit approval.

The second failure is prompt leakage. Property tickets can contain names, unit numbers, phone numbers, and pasted email threads. Redaction should happen before the prompt reaches the image service, and the audit record should retain the policy decision without copying the entire sensitive ticket into every downstream log.

The third failure is retry duplication. An image request is a side effect, so a network timeout leaves the client unsure whether the provider created an image. HTTP semantics distinguish retry-safe operations from operations whose effect may have happened already. Use an application request ID, persist the state transition, and retry only according to the provider's documented idempotency behavior; do not assume that a generic HTTP retry is harmless. The record needs enough detail to distinguish `pending`, `generated`, `denied`, and `review` without replaying a billable or irreversible action. A three-state mental model is too small here: the ambiguous timeout state is operationally different from a policy denial.

I keep one deliberately awkward test case in the eval set: a resident asks for a "before and after" image and includes a face in the attached description. The expected result is not a prettier prompt. It is a redacted, reviewable decision. Small tests like this catch more policy regressions than a gallery of attractive outputs.

## Compare the contract before comparing the picture

Score the complete workflow. A useful test matrix has ordinary maintenance diagrams, ambiguous requests, explicit disallowed content, prompt-injection text inside a ticket, missing fields, timeouts, and repeated request IDs. Record schema-validity rate, unsafe-allow rate, unnecessary-denial rate, latency, and the percentage of cases sent to review. Keep image aesthetics as a separate score.

| Decision criterion | Direct image call | Preflight plus image call |
| --- | --- | --- |
| Safety decision | External or implicit | Explicit JSON decision owned by the application |
| Missing moderation endpoint | Hard to compensate for | A chat-model gate can provide a reviewable boundary |
| Latency and token use | Lower | Higher, because there is a separate decision request |
| Failure handling | Ambiguous after a timeout | Policy state and image state can be recorded separately |

The table is a decision aid, not a ranking. A direct call can be reasonable for a tightly scoped internal tool with human review, while a preflight is more useful when tickets arrive unattended and the output feeds another system.

The trade-off is real. A chat-model preflight adds latency and token cost, and a strict schema can deny borderline requests that a human would have approved. A direct image call is simpler and may feel faster, but it leaves your application with less control over the decision boundary. I'm not sure any single threshold will fit every property portfolio; local policy, legal review, and your mileage may vary by region and tenant population.

The recommendation is not suitable when the product needs real-time, zero-review image generation or when the support team cannot operate a policy queue. In those cases, stick with a narrower image feature, a human-only workflow, or an API whose documented safety controls match the required latency. Do not choose on a single sample prompt.

## Keep the evaluation loop operational

Before shipping, pin the schema, version the policy prompt, and store a correlation ID for each decision and image request. Run the same eval set whenever the chat model, image model, safety rules, or adapter changes. Monitor denials and review volume for sudden shifts, then inspect samples rather than optimizing blindly for a lower denial rate.

Keep the notebook-to-prod path visible: the examples used to explore the classifier should become fixtures in CI, while production decisions should be replayable without exposing raw tenant data. That is where an AI application becomes maintainable. The API choice is only one input to that loop.

## Sources

- https://www.rfc-editor.org/rfc/rfc9110
- https://github.com/openai/whisper
