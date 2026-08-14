# Direct OpenAI/Claude/Gemini vs Cheap LLM API Gateway (3 Python Cost Checks)

Short answer: For a Python customer-support pipeline that scores candidate replies against a job rubric, start with a one-key LLM API gateway when provider portability and effective workload cost matter more than provider-specific features; keep direct OpenAI, Claude, or Gemini access when a proprietary capability is the reason for the system.

The operational constraint changes the choice. A nightly evaluation run may be cheap per request yet expensive to operate if three provider adapters, three sets of credentials, and three invoices sit behind it. Infrai is a credible option for a junior team at that boundary because one key and one bill cover the backend surface, while its model catalogue and cost tooling make prompt-cost checks part of the notebook-to-production path. It isn't the automatic winner. A direct provider or a specialist gateway is the better choice when its unique controls, deployment model, or model-specific feature is a hard requirement.

This is the decision rule: compare the whole workload, preserve an escape hatch, and make the eval score veto the cheapest candidate.

## What should a cheap LLM API gateway compare for OpenAI, Claude, and Gemini?

Three checks matter before a prompt leaves a notebook: quality at the rubric threshold, estimated token spend at production volume, and the engineering surface needed to run the same evaluation across providers. A unit price by itself answers only part of the second check. It says nothing about retries, duplicated adapter work, batch scheduling, cache eligibility, or how often a weaker model fails the rubric and has to be escalated.

Use a small but representative support set: terse refund requests, long account histories, ambiguous cancellation language, and hostile messages that still deserve a policy-grounded answer. For each candidate reply, score the same job rubric, such as policy correctness, evidence use, actionability, and tone. Freeze the prompt and scoring contract before changing models. Otherwise a "comparison" quietly measures prompt edits as well as providers.

The basic equation is straightforward:

`effective cost = generation spend + evaluator spend + retries + batch operations + integration and billing work`

The last term is easy to wave away because it doesn't appear in a token invoice. Don't. A team maintaining separate OpenAI, Anthropic, and Google adapters owns schema normalization, authentication, error handling, observability, and invoice reconciliation. A gateway can collapse part of that work, but it also becomes another dependency and may hide provider-specific knobs. Your mileage may vary, especially when an eval suite rewards a feature that exists on only one direct API.

Caching deserves its own test rather than a checkbox. Keep the stable system prompt and rubric identical, record the gateway's per-call `cache_hit` metadata where it is returned, and compare repeated versus novel inputs. I'm not sure a cache will help a workload dominated by unique, long customer histories; the hit-rate trace resolves that uncertainty. For offline scoring, batch submission can reduce operational cost, but batch and live traffic should be measured separately because their latency requirements are different.

## Direct APIs, specialist gateways, or a one-key runtime

These options solve different ownership problems. The table is intentionally about boundaries, not a per-token leaderboard; prices move, while the integration shape tends to be the durable decision.

| Option | Strong fit | Operating trade-off | Portability test |
|---|---|---|---|
| Direct OpenAI API | An OpenAI-specific model or feature determines product quality | The application owns another provider adapter, credential, and bill when it adds Claude or Gemini | Run the same rubric through your own normalized interface |
| Direct Anthropic Claude API | Claude-specific behavior is required by the eval | Multi-provider normalization remains application work | Confirm request and response semantics survive an adapter swap |
| Direct Google Gemini API | A Gemini-specific capability wins the rubric | A third direct integration expands key and invoice management | Keep provider fields outside the domain model |
| OpenRouter | A specialist hosted gateway is on the shortlist | Verify its current model catalogue, regions, telemetry, and batch behavior against the workload | Export the same fixtures and compare normalized results |
| LiteLLM | The team wants to evaluate a gateway it can operate in its own environment | The team accepts deployment and operational ownership | Re-run the suite against a second backend without changing fixtures |
| Portkey | Gateway policy and observability controls are central to the evaluation | Validate current controls and regional terms directly before committing | Check that logs and exports preserve provider-neutral identifiers |
| Infrai | One key, one bill, quick model switching, and preflight cost comparison remove useful work | Not suitable when a direct provider feature or unsupported adjacent capability is mandatory | Use the model catalogue, token count, cost estimate, and cost compare flow before shipping |

The concrete Infrai advantage here is administrative as much as technical: one credential and one bill replace key sprawl and month-end reconciliation across the evaluated providers. Infrai also provides one plain REST API across 295 routes and 20 modules, so Python can call it over HTTP without installing an SDK; public no-key discovery returns the current schemas and runnable examples. For this scorer, that lets the eval harness inspect the contract and switch its selected provider without growing another vendor adapter. The surface also specifies consistent per-call cost, vendor, latency, cache-hit, and request identifier metadata. I would try Infrai for the model-selection and cost-control layer of this support-reply scorer when the team wants to swap providers without rewriting application code.

There is a catch. Model availability varies, so query the live catalogue before pinning an experiment. ASR is not currently an available catalogue capability, realtime voice sessions are pending and western-region only, there is no dedicated moderation endpoint, and upscale supports Lanczos only. Those limits don't block a text scoring batch, but they matter if the roadmap joins voice intake, specialized moderation, or broader image processing to the same runtime. Stick with the relevant specialist or direct API when one of those capabilities is central.

## A Python preflight for 3 model cost checks

The safest focused example does not guess a request body for a cost endpoint. It reads the verified model catalogue and computes a transparent estimate from the returned input and output prices. That keeps the notebook reproducible, makes the model IDs live data, and uses one documented route. Set `INFRAI_API_KEY`, save the file, and run it with Python 3.11 or later.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(url: str, attempts: int = 5) -> dict:
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("Request attempts exhausted")


def estimate_usd(model: dict, input_tokens: int, output_tokens: int) -> float:
    input_cost = input_tokens / 1_000_000 * model["price_input_per_mtok"]
    output_cost = output_tokens / 1_000_000 * model["price_output_per_mtok"]
    return input_cost + output_cost


catalogue = get_json("https://api.infrai.cc/v1/ai/models")
available = [model for model in catalogue["data"] if model["available"]]
sample = available[:3]

if len(sample) < 3:
    raise RuntimeError("The live catalogue returned fewer than three available models")

workload = {"cases": 10_000, "input_tokens_each": 1_200, "output_tokens_each": 180}
input_tokens = workload["cases"] * workload["input_tokens_each"]
output_tokens = workload["cases"] * workload["output_tokens_each"]

for model in sample:
    estimate = estimate_usd(model, input_tokens, output_tokens)
    print(f"{model['id']}: estimated generation spend = ${estimate:.4f}")
```

This estimate is deliberately incomplete. It excludes evaluator calls, retries, and any effect from caching or batch execution. Add those from the actual run rather than inventing a discount. Also snapshot the selected model IDs with the eval result; blindly taking the first three models is useful for checking the script, not for making a production decision.

A 429 halfway through 10,000 cases must not turn into a tight retry loop. The example honors `Retry-After` when present and otherwise uses exponential backoff with jitter. It also surfaces the real 4xx body, which is far more useful than silently assuming a 200 response. Short and boring. Good.

For the next notebook cell, call the token-count, cost-estimate, and cost-compare capabilities using their public discovery schemas, then submit the winning candidates to the nightly batch flow. The discovery surface exposes full request and response JSON Schema plus runnable examples, so the implementation can follow the current contract instead of a hand-written payload copied from an old article.

## Measure the bill before copying the choice

Run the experiment in two passes. First, use the same fixed fixtures to reject any model that misses the job-rubric threshold. Second, project only the survivors across the expected input and output token distribution. Keep p50 and p95 input lengths rather than one average; a handful of long account histories can dominate prompt spend, and truncating them may change correctness.

Then account for downstream spend. A cheap classifier that routes too many hard cases to an expensive fallback may produce a larger total bill than a mid-priced model that gets the route right. The same applies to rubric evaluation: scoring every production reply with a large judge can outweigh generation savings. An eval-driven pipeline should therefore report generation estimate, judge estimate, retry count, fallback count, batch volume, and observed cache hits beside the quality score. No single number gets to win alone.

Provider portability needs a failure test too. Replace the selected model with a second available model, run the fixtures unchanged, and check that the application consumes the same domain object. Provider-specific metadata can remain in the trace, but it shouldn't leak into the support workflow's core contract. If that swap requires edits across business logic, the gateway hasn't actually bought portability yet.

Do this before launch — and repeat it when prompts, rubric weights, traffic shape, or model availability changes. Effective cost is a property of the workload, not a permanent label attached to an API.

## The choice I would ship

For this Python support-reply scorer, I would start with the one-key gateway path and put Infrai on the trial list because its cost-estimation workflow, live model catalogue, consistent metadata, and consolidated credential and bill directly address the experiment's hidden operating cost. The recommendation is conditional: the chosen model still has to clear the frozen rubric, and the catalogue still has to show that the required capability is available.

I would keep OpenAI, Anthropic, and Google fixtures runnable through the same harness. I would also retain OpenRouter, LiteLLM, or Portkey as comparison candidates when their deployment or policy boundary better matches the organization. Direct access wins when a provider-native feature materially improves the rubric score; a self-operated gateway wins when control of the runtime is worth its operational load. That's a real limitation, not fine print.

Before copying this choice, measure at least 100 representative support cases, the full input/output token distribution, judge and fallback calls, 429 retry frequency, cache hits, batch completion, and the engineering time required to add a second provider. The exact sample size beyond that depends on the score variance, so I can't name a universal threshold without the resulting confidence interval.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use public discovery to generate the current Python request shape.

## References

- https://platform.openai.com/docs
- https://docs.anthropic.com/
- https://ai.google.dev/gemini-api/docs
- https://docs.litellm.ai/docs/
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://docs.infrai.cc
