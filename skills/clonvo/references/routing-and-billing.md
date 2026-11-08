# Routing and billing

Canonical docs: https://clonvo.com/docs/fallbacks.md, https://clonvo.com/docs/byok.md, https://clonvo.com/docs/credits.md, https://clonvo.com/docs/plans.md

## Fallback models

Pay as you go and above (`402 plan_feature_unavailable` on Free).

```python
completion = client.chat.completions.create(
    model="anthropic/claude-sonnet-5-5",
    messages=messages,
    extra_body={"models": ["openai/gpt-6.1-sol", "google/gemini-3.8-flash"]},
)
print(completion.model, completion.model_extra["provider"])  # what answered
```

```ts
const completion = await client.chat.completions.create({
  model: "anthropic/claude-sonnet-5-5",
  messages,
  // @ts-expect-error Clonvo extension
  models: ["openai/gpt-6.1-sol", "google/gemini-3.8-flash"],
});
```

- Clonvo tries `model`, then each entry in `models` in order (up to 5), and returns the first success. You're billed at the price of the model that answered; failed attempts are never charged.
- Moves on for: `429`; `5xx`, overload, timeout, unreachable provider; credentials rejected by the provider (next credentials first, then next model); a model that can't take the request (retired, switched off, missing a needed capability such as tools or images, excluded by privacy settings, free model over its limit, no credits for it).
- Doesn't move on for a `400` from the provider (unsupported parameter, prompt longer than the context). That error returns straight away.
- Streaming: only before the first chunk.
- If everything fails, you get the last provider error, or the most actionable skip reason (`402` beats `429`, beats `403`/`503`/`404`).
- Choose fallbacks with similar capabilities (tools, structured outputs, modalities) and context length, or they'll be skipped.
- Not supported on `/v1/embeddings`; send one model per request.

## The user's own provider keys (BYOK)

Pay as you go and above. Added in the dashboard under **Provider keys**; nothing changes in code.

- For each model, Clonvo uses the organization's key for that provider first (oldest first, no rotation), then Clonvo credits.
- A key that fails with a rate limit, server error, timeout or auth error falls through to the next key, then credits, then the next fallback model, unless the key is set to **Always use this key** (then credits are never used for that provider).
- `usage.is_byok: true` means the user's key served it; `usage.cost` still shows the list price.
- Fee: none on the first $25,000 of list-price usage on own keys per month (UTC), then 4%, deducted from credits. Past the allowance with no credits: `402 byok_fee_requires_credits`.
- Some providers run only on the user's own key: `400 provider_requires_own_key` until one is added.

### provider.order

Chooses between the user's own keys and credits for one request. It accepts only `"byok"` and `"credits"`; other entries (provider names) are ignored.

| Request | Credentials used per model |
| --- | --- |
| no `provider.order` | own keys, then credits |
| `{"order": ["credits"]}` | credits, then own keys |
| `{"order": ["credits"], "allow_fallbacks": false}` | credits only |
| `{"order": ["byok"], "allow_fallbacks": false}` | own keys only; models without a key are skipped |

A combination that leaves no usable credential returns `400 no_allowed_credentials`. `allow_fallbacks` doesn't turn off `models`.

## Data collection

`provider: {"data_collection": "deny"}` excludes endpoints that may store or train on prompts for that request: free models are skipped (`404 no_endpoints_for_data_policy` if nothing is left) and paid models use their standard rather than data-sharing version.

Clonvo stores request metadata, not prompts or responses, unless an organization admin turns on retention. Free-model requests are the exception (stored up to 12 months for training).

## Cost and credits

```text
cost = (prompt − cached − cache_write) × prompt price
     + cached × cache read price
     + cache_write × cache write price
     + completion × completion price
     + per-request and web search charges, where the model has them
```

- Prices are each provider's list price on every plan. Clonvo charges a fee on credit purchases (see https://clonvo.com/pricing.md for current fees).
- `usage.cost` is on every response (and the final stream chunk).
- Before a request, Clonvo reserves its maximum cost. Without `max_tokens`, it caps output at what the balance can pay for (`finish_reason: "length"` if that cap is hit). If even that doesn't fit, or your own `max_tokens` is too high: `402 insufficient_credits`. The unused reservation is released within seconds.
- `GET /v1/credits` returns `total_credits` and `total_usage`; balance = `total_credits − total_usage`.

## Spending limits and keys

- Each API key can have a USD limit resetting daily, weekly, monthly (UTC) or never. Over it: `402 key_limit_exceeded` until the reset. `GET /v1/key` shows `limit`, `limit_remaining`, `limit_resets_at` and the `plan`.
- Keys belong to a project in an organization. Use a key per app or environment, with a limit on keys used by scripts, CI and evals.
- Rate limits: 10,000 requests a minute per key (abuse ceiling, `429 rate_limit_exceeded`, `Retry-After: 60`). Paid models have no other Clonvo-side rate limit; provider limits pass through.

## Plans

| | Free | Pay as you go | Business | Enterprise |
| --- | --- | --- | --- | --- |
| Models | `:free` only | all | all | all |
| `models`, `provider`, own keys, web tools, spending limits | no | yes | yes | yes |

Any credit purchase moves Free to Pay as you go. Details and fees: https://clonvo.com/docs/plans.md
