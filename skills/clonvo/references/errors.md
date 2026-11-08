# Errors

Errors use OpenAI's shape, so the OpenAI SDKs raise their usual exceptions.

```json
{ "error": { "message": "…", "type": "insufficient_credits", "code": "insufficient_credits", "param": null } }
```

- `message` says what went wrong and what to do. Provider errors start with the provider's name. Log it; don't show it to end users verbatim.
- `code` is the thing to branch on. `param` names the field at fault, such as `model` or `messages[0].content`.
- Model, embeddings and tool responses carry `x-clonvo-generation-id`, errors included.

Canonical docs: https://clonvo.com/docs/errors.md

## Status codes

| Status | Codes | Retry? | What to do |
| --- | --- | --- | --- |
| `400` | `invalid_json`, `invalid_request`, `model_required`, `too_many_models`, `model_not_found`, `unsupported_parameter`, `unsupported_content`, `missing_parameter`, `no_allowed_credentials`, `media_too_large`, `provider_key_rejected`, `provider_requires_own_key`, `model_not_embedding`, `embedding_model_not_chat`, `invalid_dimensions`, `invalid_tool_arguments` | No | Fix the request. Check the model id against `GET /v1/models`. |
| `401` | `missing_api_key`, `invalid_api_key`, `api_key_revoked`, `api_key_disabled`, `api_key_expired`, `project_archived` | No | Check `CLONVO_API_KEY` and the key in the dashboard. |
| `402` | `insufficient_credits`, `key_limit_exceeded`, `byok_fee_requires_credits`, `plan_feature_unavailable`, `byok_requires_paid_plan`, `project_limit_reached` | No | Credits, the key's spending limit, or the plan. On Free, every paid model returns `insufficient_credits`, and `models`/`provider` return `plan_feature_unavailable`. |
| `403` | `training_providers_not_allowed`, `free_model_terms_required` | No | Free models are off or need their terms accepted in **Settings → Privacy**. |
| `404` | `model_unavailable`, `model_retired`, `no_endpoints_for_data_policy`, `model_not_found`, `generation_not_found`, `tool_not_found`, `unknown_endpoint` | No (except `generation_not_found`, see below) | Choose another model or endpoint; add fallbacks. |
| `405` | `method_not_allowed` | No | Use the method in the `Allow` header. |
| `413` | | No | Request too large (embeddings over 4 MB). |
| `429` | `rate_limit_exceeded`, `free_model_rate_limited`, `free_model_daily_limit` | Yes | Back off; respect `Retry-After`. Add fallback models. |
| `500` | `internal_error` | Yes | Retry with backoff. |
| `502` | `upstream_error`, `upstream_unreachable`, `upstream_invalid_response` | Yes | Retry, or add fallback models. |
| `503` | `provider_unavailable`, `upstream_overloaded` | Yes | Retry later, or add fallback models. |
| `504` | `upstream_timeout` | Yes | Retry, ideally with a fallback model. |

Mid-stream errors arrive as a chunk with an `error` object and `finish_reason: "error"`; the HTTP status is already `200`.

## Retry policy

- Retry `429` and `5xx` with exponential backoff and jitter from about one second. Respect `Retry-After` when present (Clonvo sets it on its own `429`s).
- Never retry `400`, `401`, `402`, `403`, `404`: the same request fails again.
- Prefer fallback `models` over client-side retries for provider trouble: Clonvo retries on other models before answering, and failed attempts aren't charged.

```python
import openai

try:
    completion = client.chat.completions.create(model=model, messages=messages)
except openai.BadRequestError as e:       # 400: fix the request
    ...
except openai.AuthenticationError:        # 401: check the key
    ...
except openai.PermissionDeniedError:      # 403: free models need the privacy setting
    ...
except openai.NotFoundError:              # 404: model unavailable or retired
    ...
except openai.RateLimitError:             # 429: back off and retry
    ...
except openai.APIStatusError as e:
    if e.status_code == 402:              # credits, key limit, or plan: read e.code
        ...
    else:
        raise
```

## Debugging a request

```bash
# One request: what ran, on which provider, attempts, cost, latency, error
curl "https://api.clonvo.com/v1/generation?id=gen-…" -H "Authorization: Bearer $CLONVO_API_KEY"

# The key: label, spending limit, remaining, reset, plan
curl https://api.clonvo.com/v1/key -H "Authorization: Bearer $CLONVO_API_KEY"

# Organization credits: balance = total_credits - total_usage
curl https://api.clonvo.com/v1/credits -H "Authorization: Bearer $CLONVO_API_KEY"
```

- `GET /v1/generation` fields worth reading: `model` vs `requested_model`, `provider_name`, `status` (`success`, `error`, `cancelled`), `error`, `attempts` (fallbacks included), `billing_mode` (`credits`, `byok`, `free`), `total_cost` (list price) vs `usage` (actually deducted), `latency`, `time_to_first_token`.
- Usage is recorded a few seconds after a request ends: `404 generation_not_found` right after a response means retry shortly.
- You can only look up requests from your own organization.
