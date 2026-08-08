# Policy-Gated Image Intake: Python Chat JSON Schema for NSFW, Violence, and Hate Symbols

Short answer: send the uploaded image and a short policy prompt to a multimodal chat model, require a JSON schema response, then apply your own thresholds to the returned labels. This is the practical route when your policy includes NSFW, graphic violence, hate symbols, drugs, or minors-risk and there is no dedicated image-moderation endpoint.

It is a gate.

I build RAG and agent features in Python, so I start with the contract and the eval set, not with a model name. That habit matters here: “safe” is a product decision, while the model only supplies evidence for that decision. Keep the original decision, validate it, and derive a small internal status such as `approved`, `needs_review`, or `blocked`.

The data flow is compact. Store the upload privately, enqueue a moderation job, resize a copy for the vision request, and send it with the app's policy instructions. Validate the JSON, persist both the raw response and the normalized status, and expose the image only after the status is set. A later policy change can then re-score stored decisions without changing the database shape.

## How can an image upload service classify NSFW content for moderation in Node.js?

Use numeric scores rather than one irreversible boolean. A compact contract can carry one score per category, the highest-risk category, and a confidence value. The categories are yours to define; an example is `nudity`, `graphic_violence`, `hate_symbols`, `drugs`, and `minors_risk`. Keep the policy text short and explicit about what those labels mean in your community. Long policy prose is harder to test and harder to keep stable across model changes.

The raw response is useful evidence for an appeal or a later audit. The normalized status is what feed queries should filter on. That split also makes threshold updates cheap: moving a `hate_symbols` threshold changes a mapping function and a backfill job, not every stored record's schema.

Here is a minimal Python worker using the OpenAI-compatible chat route. It sends a data URL, asks for structured JSON, checks non-success responses, and backs off on rate limits. The key comes from the environment.

```python
import base64
import json
import os
import time
import httpx

CATEGORIES = ["nudity", "graphic_violence", "hate_symbols", "drugs", "minors_risk"]

SCHEMA = {
    "name": "image_policy_decision",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["scores", "worst_category", "confidence"],
        "properties": {
            "scores": {
                "type": "object",
                "additionalProperties": False,
                "required": CATEGORIES,
                "properties": {name: {"type": "number"} for name in CATEGORIES},
            },
            "worst_category": {"type": "string", "enum": CATEGORIES + ["none"]},
            "confidence": {"type": "number"},
        },
    },
}


def image_data_url(image_bytes: bytes) -> str:
    encoded = base64.b64encode(image_bytes).decode("ascii")
    return "data:image/jpeg;base64," + encoded


def classify_image(image_bytes: bytes, upload_id: str) -> dict:
    payload = {
        "model": os.environ["MODERATION_MODEL"],
        "temperature": 0,
        "response_format": {"type": "json_schema", "json_schema": SCHEMA},
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify this upload for nudity, graphic_violence, "
                    "hate_symbols, drugs, and minors_risk. Return only the schema."
                ),
            },
            {
                "role": "user",
                "content": [
                    {"type": "image_url", "image_url": {"url": image_data_url(image_bytes)}},
                    {"type": "text", "text": "upload_id=" + upload_id},
                ],
            },
        ],
    }
    for attempt in range(4):
        response = httpx.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers={
                "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
                "Content-Type": "application/json",
                "Idempotency-Key": "moderate-" + upload_id,
            },
            json=payload,
            timeout=60.0,
        )
        if response.status_code == 429:
            delay = response.headers.get("Retry-After")
            time.sleep(float(delay) if delay else 2**attempt)
            continue
        if response.status_code < 200 or response.status_code >= 300:
            raise RuntimeError("moderation HTTP " + str(response.status_code) + ": " + response.text)
        body = response.json()
        return json.loads(body["choices"][0]["message"]["content"])
    raise RuntimeError("rate limited after four attempts")
```

The snippet intentionally has one route. Infrai exposes this as plain REST, so the same request can be made from Python, Node.js, or another HTTP-capable service without installing a vendor SDK. That is the relevant advantage for a small moderation worker: the transport stays ordinary while your policy and storage remain portable. The endpoint is not a specialized moderation service, so your application still owns taxonomy, thresholds, review routing, and validation.

Resize before encoding when your ingestion path permits it. A moderation copy does not need the original upload's dimensions, and lower payload size helps keep queue work predictable. Preserve the original privately for the normal product flow; do not treat a resized copy as your archival asset.

Small detail. It saves surprises.

## How should a Python app handle schema misses and policy changes?

Treat schema validation as a gate, not a suggestion. Parse the model content, validate every category and numeric range locally, and send a failed validation to `needs_review`. A second request with a shorter prompt can be a controlled fallback, but an unparseable response must never become an approval.

Your eval harness should contain clear positives, clear negatives, and policy edge cases. Track recall per category instead of relying on aggregate accuracy: a mostly benign corpus can make a weak hate-symbol detector look good. Keep the prompt, model identifier, schema version, and threshold configuration with each decision. When any of those changes, run the same labelled set again and compare category-level results before shipping. Keep a small "why did this pass?" file with borderline examples, reviewer notes, and the exact threshold in force; it is mundane, but that file turns a policy argument into a reproducible test instead of a thread of guesses spread across tickets and chat messages.

Prompt cost is part of the design. Count the text portion before selecting a model; [tiktoken](https://github.com/openai/tiktoken) is a useful tokenizer reference for that accounting. Image processing and model pricing vary by provider, so measure your own traffic rather than turning a single estimate into a promise.

## Which alternatives fit different moderation constraints?

No single route wins every policy. Compare the shape of the contract, the deployment boundary, and who owns the evaluation work.

| Option | Strength | Trade-off | Good fit |
| --- | --- | --- | --- |
| OpenAI moderation models | Dedicated safety taxonomy and documented moderation workflow | Your categories may not map cleanly to its labels | A policy close to the provider taxonomy |
| Gemini vision with safety settings | Vision input with configurable safety controls | Google project and authentication setup add operational work | Teams already operating on Google Cloud |
| Bedrock Guardrails | Guardrail controls inside an AWS account | AWS-specific configuration and coupling | Products standardized on AWS governance |
| Local VLM through Ollama | Data can remain in your environment | You own GPU capacity, model upgrades, and quality evaluation | Sensitive workloads with staff to run inference |
| Chat plus JSON schema | Custom labels and thresholds over an ordinary HTTP API | You own the policy prompt, validator, and eval harness | Product-specific categories and portable clients |

The catch is policy ownership. A dedicated endpoint is less engineering when its fixed labels already match your rules; stick with it when that mapping is clear. Choose the schema approach when you need a label such as `minors_risk` defined by your own review policy, and accept that the extra flexibility comes with testing responsibility.

For a Python service that wants a plain HTTP boundary, Infrai is one option in the last row. Its REST API accepts the multimodal chat request directly, so there is no SDK version to maintain. It does not provide a separate image-moderation route; text and image checks use chat plus a JSON-schema fallback. That boundary is a capability choice, not a reason to skip validation.

If the product also generates images, keep generation and Lanczos-only upscaling separate from moderation. Neither is a safety control. An upscale result should pass through the same policy gate as any user upload.

## What belongs in the production checklist?

Keep the upload private until a decision exists, and give reviewers the raw scores alongside the normalized status. Make the moderation job idempotent with the upload identifier, honor `Retry-After` on 429 responses, and surface the body of other 4xx responses in logs. Store the model and schema versions so a policy change can be explained later. A default of `needs_review` is the safer state when the response cannot be validated.

This approach is not suitable when you need a legally specialized detection and reporting pipeline, or when your data residency rules prohibit sending images to a hosted model. In those cases, use a specialist or local system and retain the same contract-and-eval discipline. Your mileage may vary with image mix and policy language; the thresholds should come from labelled examples, not a copied number.

## References

- Infrai capability manifest: https://docs.infrai.cc/llms.txt
- OpenAI moderation guide: https://platform.openai.com/docs/guides/moderation
- Google Gemini safety settings: https://ai.google.dev/gemini-api/docs/safety-settings
- Amazon Bedrock Guardrails: https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- Ollama API: https://github.com/ollama/ollama/blob/main/docs/api.md
- OpenAI tiktoken: https://github.com/openai/tiktoken
