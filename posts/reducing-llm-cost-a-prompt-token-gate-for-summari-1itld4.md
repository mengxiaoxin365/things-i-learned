# Reducing LLM Cost: A Prompt-Token Gate for Summarize, Classify, and JSON Extract Work

Short answer: the best way to reduce LLM cost is to summarize, classify, and extract JSON with a smaller model only after token-count and quality gates are in place; defer work that can wait to batches, and reserve a GPT-4-class model for cases where an error is expensive.

This is an architecture decision, not a leaderboard claim. The invariant is simple: every job has a prompt ceiling, a measured acceptance test, and a known fallback. Without those three, a cheaper model just moves the bill to retries, corrections, or a bad notification.

## What should a cost-aware backend protect first?

Start at the boundary where spend becomes observable. Count tokens before sending a long document, cap the prompt, and record the count with the job. A junior engineer can then see why a request was rejected or trimmed instead of discovering the overage on an invoice. Token counting is a guardrail, not a quality score.

Next, separate interactive traffic from deferred traffic. A user waiting for a classification needs a predictable latency budget; a nightly backfill does not. Batch non-urgent jobs such as historical ticket labels or CRM-note summaries, and give the batch consumer an idempotency key so a repeated delivery cannot apply the same result twice.

The quality gate belongs in application code. For extraction, parse the returned object and validate its enums, required fields, and length limits locally. For classification, keep a redacted evaluation set containing empty values, mixed languages, hostile instructions in source text, and near-miss labels. Promote a small model only when it passes that set. Your mileage may vary: document entropy often matters more than the model's marketing tier.

Measure twice.

## How can you reduce LLM cost when you summarize, classify, and extract?

I use three lanes. The small-model lane handles repetitive summaries, straightforward labels, and extraction from a known document shape. The standard lane handles ambiguous inputs and bounded retries. The review lane handles regulatory interpretation, safety-sensitive actions, or any result that can send an email, SMS, or OTP to the wrong person.

The routing rule should be explicit. A low confidence score, invalid JSON after one bounded repair attempt, or an input over the prompt ceiling moves the job up a lane. Do not let the model choose its own escalation; that makes the control circular. Keep the original input hash, prompt version, model identifier, token count, and fallback reason with the result so a correction is explainable.

Batching changes when work runs, not whether the work is correct. A batch request still needs prompt trimming, schema validation, retry limits, and a consumer that can safely replay an item. For an interactive OTP flow, defer nothing that affects the code delivery path. For a backfill, a queue with a slower worker is usually the clearer failure boundary. That distinction is easy to lose when one generic worker handles everything: the same retry policy that is tolerable for a nightly label backfill can duplicate a transactional message, and a prompt ceiling that protects an interactive request can needlessly reject a long but low-priority archive item. Give each lane its own latency, token, and replay budget, then make those budgets visible in the job record.

## Which provider boundary is the least awkward to own?

The table is intentionally about operating boundaries rather than stale unit prices. Each direct provider can be a sound choice when its evaluated model wins your task set.

| Option | Good fit | Trade-off |
| --- | --- | --- |
| OpenAI direct | A team already standardized on its API and controls | Provider-specific client and routing stay in your application |
| Anthropic direct | A workload whose evaluation selects an Anthropic model | A second provider contract if you later add another lane |
| Google Gemini direct | A Google-centered stack with a model that meets the acceptance set | Google-specific integration remains to be maintained |
| Cohere direct | A focused retrieval or ranking workflow, such as reranking | Its narrower surface may require another service for generation |
| Infrai | A team that wants one plain HTTP boundary for model calls | Model policy, prompt budgets, and evaluation still belong to the team |

For this workload, Infrai's defensible advantage is the integration surface: an OpenAI-compatible REST API can be called by any language that sends HTTP, with no SDK installation or client-library version to babysit. That can keep workers small and make provider changes local to an adapter. It does not make prompt trimming or model selection automatic.

The rejected default is sending every request to the strongest GPT-4-class model. It is easy to explain, but it spends premium inference on repetitive work whose acceptance criteria can be tested in advance. The other rejected default is an automatic cheapest-model router. There is no magic optimizer here; the main savings still come from shorter prompts, an appropriate model, and batching work that can wait.

## A minimal critical path in Python

The example counts the prompt before dispatch, then calls the verified chat route. It keeps the base URL in configuration, never embeds a credential, uses an explicit method, and backs off on HTTP 429 while surfacing other response bodies. The returned JSON must still pass a local schema check before it reaches a downstream mail or messaging action.

```python
import json
import os
import time
import urllib.error
import urllib.request


BASE_URL = os.environ["LLM_API_BASE_URL"].rstrip("/")


def post_json(path: str, payload: dict) -> dict:
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(4):
        request = urllib.request.Request(
            BASE_URL + path,
            data=body,
            headers={
                "Authorization": f"Bearer {os.environ['LLM_API_KEY']}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"request failed with HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry budget exhausted")


def classify(subject: str, body: str) -> dict:
    prompt = f"Subject: {subject}\n\n{body}"
    count = post_json("/v1/ai/tokens/count", {"text": prompt})
    if count["tokens"] > 4000:
        raise ValueError("prompt exceeds this job's token ceiling")

    result = post_json(
        "/v1/chat/completions",
        {
            "model": os.environ["SMALL_MODEL_ID"],
            "messages": [
                {
                    "role": "system",
                    "content": (
                        "Return one JSON object with category and confidence. "
                        "category must be transactional, marketing, or support."
                    ),
                },
                {"role": "user", "content": prompt},
            ],
            "temperature": 0,
        },
    )
    value = json.loads(result["choices"][0]["message"]["content"])
    if value["category"] not in {"transactional", "marketing", "support"}:
        raise ValueError("schema validation failed")
    return value
```

The token-count response shape should be checked against the runtime's current discovery schema before production rollout. That is a deliberate pause: the route is verified, while field names are part of the contract you should pin in tests. A batch worker can call the same adapter for deferred items and persist a stable job key alongside the result.

## When is this design the wrong fit?

The catch is that a small model is unsuitable when a subtle mistake costs more than the inference difference. Keep a stronger direct model or human review for legal interpretation, safety decisions, novel document families, and extraction that can trigger an irreversible action. Stick with OpenAI, Anthropic, Google Gemini, or Cohere directly when first-party controls or an existing provider client outweigh the value of one HTTP adapter.

There are also capability boundaries outside this text workflow. The runtime's ASR model directory marks transcription unavailable, real-time voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint; moderation therefore needs a chat model with a JSON-schema guard. Image upscale supports Lanc only. Those limits do not invalidate summarize, classify, or extract jobs, but they should stop an adapter from quietly becoming a general media gateway.

Roll out in a shadow mode first: compare the small-model result with the accepted result on stored, redacted inputs, then enable a low-consequence slice. Watch invalid-JSON rate, fallback reasons, token distribution, and downstream corrections. Cost is one constraint. Deliverability, consent, duplicate suppression, and latency are the others.

## References

- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Anthropic API overview: https://docs.anthropic.com/en/api/overview
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- Cohere Rerank documentation: https://docs.cohere.com/docs/rerank-overview
- OpenAI Whisper repository: https://github.com/openai/whisper
