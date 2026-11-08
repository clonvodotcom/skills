# Models

Canonical docs: https://clonvo.com/docs/models.md and https://clonvo.com/docs/other-endpoints.md#get-v1-models

## Model ids

Canonical ids are `author/slug`:

| Provider | Author |
| --- | --- |
| OpenAI | `openai` |
| Anthropic | `anthropic` |
| Google (Gemini) | `google` |
| xAI (Grok) | `x-ai` |
| DeepSeek | `deepseek` |
| Mistral | `mistralai` |
| Moonshot AI (Kimi) | `moonshotai` |
| Meta (Muse) | `meta` |

Authors match OpenRouter's, so OpenRouter ids usually work unchanged. Clonvo also resolves author aliases (`xai`, `gemini`, `mistral`, `moonshot`, `kimi`), bare slugs (`grok-4-7`), `.` for `-` (`claude-sonnet-5.5`), and any letter case. Use the canonical id in code: it's what the catalogue, activity and responses show.

| Status | Code | Meaning |
| --- | --- | --- |
| `400` | `model_not_found` | No such model. One unknown id in `models` fails the whole request. |
| `404` | `model_retired` | Past the provider's retirement date. Fallbacks skip it. |
| `404` | `model_unavailable` | Temporarily switched off. Fallbacks skip it. |
| `400` | `provider_requires_own_key` | The provider isn't on Clonvo credits yet; the user must add their own key for it. |

## Free models

Ids ending in `:free` run on OpenRouter's free endpoints at $0.

- An owner or admin must turn on **Use free models** in **Settings → Privacy** and accept the data terms. Otherwise: `403 training_providers_not_allowed` (or `403 free_model_terms_required` when the terms changed).
- Prompts and responses may be logged by the providers and are stored by Clonvo for up to 12 months for training. Never send confidential, personal or regulated data.
- Limits per organization: 20 requests a minute, 50 a day (1,000 a day once it has bought $10 of credits in total). Over them: `429 free_model_rate_limited` / `429 free_model_daily_limit` with `Retry-After`.
- Aliases don't apply. Copy the id exactly as listed; the list changes as OpenRouter adds and removes models:

```bash
curl -s https://api.clonvo.com/v1/models | jq -r '.data[].id | select(endswith(":free"))'
```

- `provider.data_collection: "deny"` excludes free models (and data-sharing versions of paid models) for one request.

## GET /v1/models

Public (no key), OpenRouter format, cached up to a minute. Prices are USD **per token**, as strings: multiply by 1,000,000 for the per-1M price.

```json
{
  "id": "anthropic/claude-sonnet-5-5",
  "name": "Claude Sonnet 5.5",
  "created": 1790553600,
  "context_length": 1000000,
  "architecture": { "input_modalities": ["text", "image"], "output_modalities": ["text"] },
  "pricing": { "prompt": "0.000002", "completion": "0.00001", "input_cache_read": "0.0000001", "input_cache_write": "0.0000025", "request": "0", "web_search": "0.01" },
  "top_provider": { "context_length": 1000000, "max_completion_tokens": 128000 },
  "supported_parameters": ["max_tokens", "reasoning", "tool_choice", "tools", "response_format", "structured_outputs", "include_reasoning"]
}
```

- Chat models only by default. `?output_modalities=embeddings` (or `all`) lists embedding models; so does `GET /v1/embeddings/models`.
- `GET /v1/models/{author}/{slug}` returns one model (`404 model_not_found` if unknown).
- Retired and switched-off models aren't listed. Free models are listed with zero prices.
- Filter client-side; the endpoint has no capability filters besides `output_modalities`.

## Choosing a model

| Need | Check |
| --- | --- |
| Tool calling | `supported_parameters` has `tools` |
| JSON Schema output | `supported_parameters` has `structured_outputs` |
| Reasoning control | `supported_parameters` has `reasoning` |
| Images in | `architecture.input_modalities` has `image` |
| PDFs in | `architecture.input_modalities` has `file` |
| Audio in | `architecture.input_modalities` has `audio` |
| Long inputs | `context_length` |
| Long outputs | `top_provider.max_completion_tokens` |
| Cost | `pricing.prompt` and `pricing.completion` (× 1,000,000 for per-1M) |

```bash
# Cheapest paid models that support tools and images, per-1M prices
curl -s https://api.clonvo.com/v1/models | jq -r '
  .data[]
  | select((.supported_parameters | index("tools")) and (.architecture.input_modalities | index("image")))
  | select((.pricing.prompt | tonumber) > 0)
  | [(.pricing.prompt | tonumber*1e6*1e4|round/1e4), (.pricing.completion | tonumber*1e6*1e4|round/1e4), .id] | @tsv' | sort -n | head
```

The `clonvo-models` skill has more recipes.

## Peak-hour prices

A few models cost more at busy times. Their top-level `pricing` is the off-peak price and `pricing.overrides` lists weekly UTC windows (`utc_days`, `utc_start`/`utc_end` as HHMM). A request is charged the price in effect when it reaches Clonvo, and `usage.cost` reports it.

## Deprecations

When a provider announces a retirement, the model's catalogue page (https://clonvo.com/models/<author>/<slug>.md) shows the date. Move to a newer model before it; add fallbacks so a retirement doesn't break production.
