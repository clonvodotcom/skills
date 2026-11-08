---
name: clonvo
description: "Call OpenAI, Anthropic, Google Gemini, xAI Grok, DeepSeek, Mistral, Kimi and Meta models through Clonvo's one OpenAI-compatible API, with fallback models and web_search/web_scrape tools for agents (REST or MCP). Use when the user mentions Clonvo, CLONVO_API_KEY, api.clonvo.com or a sk-cv-v1- key, or wants one API key for many LLM providers, model fallbacks, or web search and scraping tools in their app or agent."
---

# Clonvo

Clonvo is an OpenAI-compatible API for models from OpenAI, Anthropic, Google, xAI, DeepSeek, Mistral, Moonshot (Kimi) and Meta, plus `web_search` and `web_scrape` tools for agents. One API key covers every model; switching models means changing the `model` string.

- Base URL: `https://api.clonvo.com/v1`
- Auth: `Authorization: Bearer $CLONVO_API_KEY` (keys look like `sk-cv-v1-` followed by 64 hex characters)
- Docs for agents: https://clonvo.com/llms.txt (index) and https://clonvo.com/llms-full.txt (everything). Every docs page is Markdown at its URL plus `.md`, e.g. https://clonvo.com/docs/fallbacks.md
- Live model list with prices and capabilities, no key needed: `GET https://api.clonvo.com/v1/models`

Read the docs page for anything this file doesn't cover. Don't guess model ids, prices or limits: they change, and the live endpoints are the source of truth.

## Setup

1. The user signs up at https://clonvo.com, creates a key under **API keys** (shown once), and sets it as an environment variable:
   ```bash
   export CLONVO_API_KEY="sk-cv-v1-…"
   ```
2. Never hard-code the key, commit it, or put it in browser or mobile code. Read it from the environment.
3. Paid models need credits. On the Free plan, paid models return `402 insufficient_credits`; free models (ids ending in `:free`) work once an owner or admin turns on **Use free models** in **Settings → Privacy**.

## Make a request

Use the official OpenAI SDK with Clonvo's base URL. Nothing else changes.

```python
import os
from openai import OpenAI

client = OpenAI(base_url="https://api.clonvo.com/v1", api_key=os.environ["CLONVO_API_KEY"])

completion = client.chat.completions.create(
    model="anthropic/claude-sonnet-5-5",
    messages=[{"role": "user", "content": "Summarise this ticket…"}],
)
print(completion.choices[0].message.content)
```

```ts
import OpenAI from "openai";

const client = new OpenAI({ baseURL: "https://api.clonvo.com/v1", apiKey: process.env.CLONVO_API_KEY });

const completion = await client.chat.completions.create({
  model: "anthropic/claude-sonnet-5-5",
  messages: [{ role: "user", content: "Summarise this ticket…" }],
});
```

Streaming, tool calling, structured outputs (`response_format`), reasoning (`reasoning_effort`), images and files work with the same request shape for every provider. `POST /v1/responses` (OpenAI's Responses API, stateless) and `POST /v1/embeddings` are also available.

The response is a standard Chat Completions object, plus:
- `model`: the canonical id of the model that answered
- `provider`: the provider that served it
- `usage.cost`: the cost in USD at the provider's list price

## Model ids

- Format: `author/slug`, for example `anthropic/claude-sonnet-5-5`, `openai/gpt-6.1-sol`, `google/gemini-3.8-flash`, `x-ai/grok-4-7`. These are examples; get current ids from `GET /v1/models`.
- Authors: `openai`, `anthropic`, `google`, `x-ai`, `deepseek`, `mistralai`, `moonshotai`, `meta`. They match OpenRouter's, so OpenRouter ids usually work unchanged.
- Clonvo also accepts author aliases (`xai`, `gemini`, `mistral`, `moonshot`, `kimi`), bare slugs (`grok-4-7`), `.` for `-`, and any letter case. Use the canonical id in code.
- `:free` ids route to OpenRouter's free endpoints, with low rate limits and their own data terms. List them with:
  ```bash
  curl -s https://api.clonvo.com/v1/models | jq -r '.data[].id | select(endswith(":free"))'
  ```
- An unknown id returns `400 model_not_found`. A retired model returns `404 model_retired`.

## Fallback models

Pay as you go and above. Add `models` (up to 5) and Clonvo tries `model` first, then each fallback in order, on rate limits (`429`), server errors, timeouts and unavailable models. Invalid requests (`400`) are returned straight away and not retried. Only the attempt that answered is charged.

```python
completion = client.chat.completions.create(
    model="anthropic/claude-sonnet-5-5",
    messages=messages,
    extra_body={"models": ["openai/gpt-6.1-sol", "google/gemini-3.8-flash"]},
)
print(completion.model)  # the model that answered
```

In TypeScript, pass `models` in the request body (the OpenAI SDK's types don't include it, so add `// @ts-expect-error Clonvo extension`).

The user's own provider keys, added under **Provider keys**, are used before credits. `provider: { order: ["byok"] | ["credits"], allow_fallbacks: false }` restricts one request to one kind. Details: https://clonvo.com/docs/fallbacks.md

## Web tools for agents

Pay as you go and above. `web_search` finds pages; `web_scrape` reads one page as Markdown or text. They run on the user's own Firecrawl, Tavily, Exa, Parallel or TinyFish key if added, otherwise on Clonvo credits.

**Over MCP** (Streamable HTTP, API key in the header; no OAuth):

```bash
claude mcp add --transport http clonvo https://api.clonvo.com/mcp \
  --header "Authorization: Bearer $CLONVO_API_KEY"
```

For a shared project config, commit this `.mcp.json` and have each person set `CLONVO_API_KEY` themselves:

```json
{
  "mcpServers": {
    "clonvo": {
      "type": "http",
      "url": "https://api.clonvo.com/mcp",
      "headers": { "Authorization": "Bearer ${CLONVO_API_KEY}" }
    }
  }
}
```

Cursor, the OpenAI Agents SDK and the AI SDK setups are at https://clonvo.com/docs/mcp.md

**Over REST**:

```bash
curl https://api.clonvo.com/v1/tools/web_search \
  -H "Authorization: Bearer $CLONVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"arguments": {"query": "PostgreSQL 18 release notes", "max_results": 3}}'
```

- `GET /v1/tools` returns each tool's JSON Schema (`input_schema`), which is a valid function `parameters` object for tool calling.
- `web_search` arguments: `query` (required), `max_results` (1–20), `include_domains`, `exclude_domains`, `time_range` (`day`/`week`/`month`/`year`), `include_content`, `max_content_chars`.
- `web_scrape` arguments: `url` (required), `format` (`markdown`/`text`), `only_main_content`, `max_chars`.
- Results are in `result`; cost is in `usage`.

Details: https://clonvo.com/docs/tools.md

## Errors

Errors use OpenAI's shape (`error.message`, `type`, `code`, `param`), so the OpenAI SDKs raise their usual exceptions. Model, embeddings and tool responses carry an `x-clonvo-generation-id` header, errors included, for looking a request up with `GET /v1/generation`.

| Status | Meaning | What to do |
| --- | --- | --- |
| `400` | Invalid request: unknown model id, unsupported parameter or content | Fix the request; check the id with `GET /v1/models` |
| `401` | Missing, invalid, revoked or expired key | Check `CLONVO_API_KEY` |
| `402` | Out of credits, the key's spending limit, or a feature not on the plan (`plan_feature_unavailable` for `models`/`provider` on Free) | The user buys credits or raises the key's limit; read `error.code` |
| `403` | Free models are off, or their terms need accepting | An owner or admin enables them in **Settings → Privacy** |
| `404` | Model retired or temporarily unavailable | Pick another model, or add fallbacks |
| `429` | Rate limited | Back off and retry, or add fallbacks |
| `5xx` | Provider error, overload or timeout | Retry, ideally with fallback models |

Full list: https://clonvo.com/docs/errors.md

## Pricing

Tokens cost the provider's list price on every plan; Clonvo charges a fee on credit purchases. Current plans and fees: https://clonvo.com/pricing.md. Per-model prices: `GET /v1/models` or https://clonvo.com/models.md
