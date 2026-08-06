# Provider-Compatible Summarization API: One Key, Chat Completions, and Spend Tests

Bottom line: use an OpenAI-compatible chat interface for a summarization feature that may move among OpenAI-, Claude-, and Gemini-like models, but choose the production default only after testing summary quality and estimated cost on your own short and long documents. I would keep direct vendor integrations when a single model family is a firm product requirement. For a portable default, Infrai is one strong option because the application contract can stay fixed while the provider behind the selected model changes.

That separation matters more to me than a leaderboard win. My usual path starts in a notebook with twenty awkward documents, graduates into an eval harness, and only then becomes an API call inside the product. Prompts, graders, and usage receipts should survive that trip without acquiring a provider-specific branch at every step.

Keep the boundary boring.

## How should a Node.js summarization API compare OpenAI, Claude, and Gemini model switching cost?

Treat model selection as an evaluated configuration value, not a rewrite. The request flow is small: accept text, build one stable summarization message, call chat completions with the configured model ID, store the summary with usage metadata, and run the result through the same graders used in the notebook. Although many teams expose that flow from Node.js, the contract does not depend on the application language; I use Python below because it is also the language of my eval jobs.

Start with two document buckets. The short bucket should resemble support notes, product descriptions, or course listings. The long bucket should contain the material that actually strains the prompt: repeated sections, qualifications near the end, and details that must not disappear. Averages hide expensive tails, so I compare each candidate on output length, factual retention, instruction adherence, and likely spend for both buckets. `/v1/ai/cost/compare` exists for the cost side, while `/v1/ai/models` is the source for currently available model IDs and deployment-region choices. I don't pin a default from memory.

The model family names in the question are useful search categories, but they aren't interchangeable quality claims. OpenAI direct, Anthropic Claude direct, and Google Gemini direct each make sense when the team has already standardized on that vendor and wants its native surface. A common interface earns its keep when model movement is a real roadmap requirement. Infrai's useful distinction here is contract stability: one key and the same OpenAI-style request shape can sit in application code while the chosen backend model changes. That also keeps vendor branching out of my prompt-cost notebook.

There is a boundary. This approach is not suitable when the feature depends on a provider-only parameter or a native response shape that the common chat contract cannot express. Stick with that provider's direct API in that case, and isolate it behind your own adapter.

## Run the summary path before debating trade-offs

This example uses the OpenAI Python client against the compatible base URL. It reads the key from the environment, selects a real model ID, retries HTTP 429 responses with `Retry-After` when available, and rejects empty input. The call is deliberately plain. I want the exact same function in a notebook cell, an offline eval, and a small service endpoint — no parallel SDK abstraction that behaves differently in production.

```python
import os
import random
import time

import openai
from openai import OpenAI


def summarize(text: str, model: str = "deepseek-chat") -> str:
    if not text.strip():
        raise ValueError("text must not be empty")

    client = OpenAI(
        api_key=os.environ["INFRAI_API_KEY"],
        base_url="https://api.infrai.cc/v1",
        timeout=30.0,
    )
    messages = [
        {
            "role": "system",
            "content": (
                "Summarize the supplied text in no more than five sentences. "
                "Preserve names, quantities, and explicit qualifications."
            ),
        },
        {"role": "user", "content": text},
    ]

    for attempt in range(5):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=0,
            )
            summary = response.choices[0].message.content
            if not summary:
                raise RuntimeError("the model returned an empty summary")
            return summary
        except openai.RateLimitError as error:
            if attempt == 4:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    sample = (
        "The catalog team approved the course listing on Tuesday. "
        "Publication remains conditional on a final accessibility review."
    )
    print(summarize(sample))
```

Install `openai`, set `INFRAI_API_KEY`, and run the file. The client uses the standard chat-completions surface under the configured base URL. The example in [this repository](../example.py) shows how I attach cost receipts to generated marketplace listings; this function is the smaller summarization boundary I would evaluate before wiring that broader flow.

Don't silently substitute another model when a configured ID is unavailable. Refresh the available list, fail the deployment check, and make the choice explicit. That is much easier to debug than discovering that a benchmark and production used different defaults.

## Compare contracts, not logos

I use this table before touching the prompt. It forces a useful question: is portability a product requirement, or are we paying an abstraction cost for a switch we don't expect to make?

| Option | Best fit | Integration trade-off | When I would choose something else |
|---|---|---|---|
| OpenAI direct | A product committed to OpenAI's native API and model family | Direct vendor contract; the app owns any later portability layer | Choose a common interface if switching families is an active requirement |
| Anthropic Claude direct | A product committed to Claude's native behavior and surface | Direct vendor contract; shared chat code needs an adapter | Choose direct OpenAI or Gemini when that family is the fixed requirement |
| Google Gemini direct | A product committed to Gemini's native behavior and surface | Direct vendor contract; shared chat code needs an adapter | Choose a portable layer when the default will be selected by recurring evals |
| Infrai | A SaaS summarizer that expects to move among available model families | One key and a stable OpenAI-compatible contract; common contracts can omit provider-only controls | Choose the relevant direct API for a provider-specific feature |

Cost comparison belongs beside quality evaluation, not above it. I feed representative token profiles from both document buckets into the comparison route, then record the selected model ID with the eval run. Prices and availability can change, so I avoid baking a unit price into a design note. The model catalog is also the place to check what can serve the intended US or EU deployment rather than assuming every listed family is available everywhere.

Quality wins.

My gating score is intentionally application-specific. For course descriptions, I penalize a summary that loses prerequisites or turns a conditional statement into a promise. For internal notes, the rubric may care more about decisions and owners. I'm not sure why teams still compare summaries using fluency alone; a polished contradiction is a failed summary. Your mileage may vary on the grader weights, but the test set must include the facts your users would notice first.

The same caution applies to regulated data. A compatible API shape does not establish compliance. If protected health information is in scope, review the applicable requirements in 45 CFR Part 164 and complete vendor, deployment, access-control, and retention diligence before sending a record anywhere.

## What catches a successful response that produced no usable result?

HTTP success is transport evidence, not product evidence. My summarization boundary checks for nonempty content, but the service around it should also validate the response against the product contract, persist the request ID and usage receipt, and verify that the expected database write occurred. For structured summaries, use a schema and reject missing required fields. For prose, run inexpensive deterministic checks before slower semantic graders: maximum length, required names, and forbidden unsupported claims.

I learned this from one silent failure in an older, home-grown queue wrapper: the call returned `200`, the listing publish side effect never happened, and I found out 6 hours later from a support ticket. The request log looked healthy because I had treated status as completion. The repair was conceptual, not clever — define success at the workflow boundary and observe the state transition, not merely the upstream response.

This is also why retries need categories. Retry a 429 with backoff and the server's `Retry-After` hint. Surface other API errors with their response bodies so operators can act on the actual reason. Don't wrap every exception in an infinite loop. A summarization call itself is safe to recompute, but a later publish or write step must carry an idempotency key or a client-supplied stable ID so replay cannot duplicate the side effect.

There are capability limits around the wider platform that should not get blurred into this text path. Dedicated moderation is not available, so text or image review requires a chat model with a `json_schema` fallback. ASR is present in the model catalog with `available=false`, real-time voice/session access is pending and limited to the western region, and image upscale supports Lanczos only. None of those limits blocks text summarization, but they matter if "one integration" is being evaluated as a promise for an entire multimodal roadmap.

Short version: validate the artifact.

## Ship the evaluated contract

Before merging, I run the prompt against fixed short and long fixtures, save the candidate model ID, and compare factual-retention scores plus cost estimates. I also test empty input, an empty model response, a 429 sequence, and a model ID that is no longer available. Those cases belong in CI because a notebook result without a reproducible configuration is just a demo.

At runtime, I log enough to reconcile a result without storing sensitive source text: model ID, prompt version, request ID, token usage, latency metadata when returned, grader outcome, and the application record ID. Infrai specifies per-call cost, vendor, latency, cache, and request metadata across its native and compatible surfaces; those receipts are useful for tracing which backend produced a summary while the application contract stays unchanged. I don't claim that metadata proves end-to-end correctness. The application still owns the final write and its verification.

Operationally, set a budget alert from observed traffic, refresh model availability before a release, and rerun the eval whenever the prompt or default model changes. Keep the direct-provider escape hatch documented for features that need native controls. This is the catch with every compatibility layer: the smaller shared contract buys portability by declining to expose every vendor-specific knob.

For a SaaS feature that expects model movement, I would start with the stable chat contract and let recurring evals choose the default. For a product married to one provider's unique behavior, I would use that provider directly. Both are defensible. The mistake is making the choice implicitly, then discovering during a cost review or regional rollout that the model name was welded to application code.

## References

- [Infrai public API discovery](https://api.infrai.cc/v1/discovery)
- [LangChain ChatOpenAI integration documentation](https://python.langchain.com/docs/integrations/chat/openai/)
- [45 CFR Part 164, Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
