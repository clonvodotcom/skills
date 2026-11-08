---
name: clonvo
description: "Build apps and agents on Clonvo, one OpenAI-compatible API for OpenAI, Anthropic, Google Gemini, xAI Grok, DeepSeek, Mistral, Kimi and Meta models: chat completions, the Responses API, embeddings, streaming, tool calling, structured outputs, reasoning, images and PDFs, fallback models, bring-your-own-key routing, and web_search/web_scrape tools over REST or MCP. Use when writing or debugging code that calls Clonvo, when the user mentions Clonvo, CLONVO_API_KEY, api.clonvo.com or an sk-cv-v1- key, or wants one API key for many LLM providers with fallbacks."
metadata:
  author: Clonvo
  version: "1.0.0"
  docs: "https://clonvo.com/llms.txt"
---

# Build with Clonvo

Clonvo is an OpenAI-compatible API in front of OpenAI, Anthropic, Google (Gemini), xAI (Grok), DeepSeek, Mistral, Moonshot (Kimi) and Meta (Muse) models. Code written for the OpenAI SDK works by changing the base URL, the key and the model id. Clonvo translates requests for providers that don't speak OpenAI's format and returns OpenAI-shaped responses.

## Scope

- Chat Completions: `POST /v1/chat/completions` (the default for new work)
- Responses API: `POST /v1/responses`, stateless; also runs the OpenAI Agents SDK
- Embeddings: `POST /v1/embeddings`
- Web tools for agents: `web_search` and `web_scrape`, over REST (`POST /v1/tools/{id}`) or MCP (`https://api.clonvo.com/mcp`)
- Read-only endpoints: `GET /v1/models` (public), `GET /v1/key`, `GET /v1/credits`, `GET /v1/generation?id=`

Not available: OpenAI's Images, Audio, Files, Batch and Assistants APIs (they return `404 unknown_endpoint`).

## Setup

```bash
export CLONVO_API_KEY="sk-cv-v1-…"   # created under API keys in the Clonvo dashboard; shown once
```

- Base URL: `https://api.clonvo.com/v1`
- Auth: `Authorization: Bearer $CLONVO_API_KEY` (`X-Api-Key` also works)
- Read the key from the environment. Never hard-code it, commit it, or ship it in browser or mobile code; call Clonvo from a server.
- Plans gate features. On Free, only `:free` models work, and `models` or `provider` in a request return `402 plan_feature_unavailable`. Any credit purchase moves the organization to Pay as you go, which unlocks every model, fallbacks, your own provider keys and the web tools.

## The recommended request

Use the official OpenAI SDK with Clonvo's base URL, a canonical model id, and nothing else unless the task needs it:

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

```bash
curl https://api.clonvo.com/v1/chat/completions \
  -H "Authorization: Bearer $CLONVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "anthropic/claude-sonnet-5-5", "messages": [{"role": "user", "content": "Summarise this ticket…"}]}'
```

The response is a standard Chat Completions object plus three things worth using:

- `model`: the canonical id of the model that answered (it can differ from what you sent: aliases resolve, and a fallback may answer)
- `provider`: the provider that served it
- `usage.cost`: the cost in USD at list price (`usage.is_byok` is `true` when the user's own provider key served it)

The id `gen-…` is also in the `x-clonvo-generation-id` header, for `GET /v1/generation?id=`.

Model ids in these examples are real today, but the catalogue changes. Get current ids from `GET https://api.clonvo.com/v1/models` (no key needed) or the `clonvo-models` skill. Never invent an id.

## Decide what to use

1. Text or multimodal generation in an app: Chat Completions. See [references/chat-completions.md](references/chat-completions.md).
2. Code already on the Responses API or the OpenAI Agents SDK: `/v1/responses`, sending the whole conversation every turn. See [references/responses-api.md](references/responses-api.md).
3. A request must survive a provider outage or rate limit: add fallback `models`. See [references/routing-and-billing.md](references/routing-and-billing.md).
4. Picking a model by price, context, or capability (tools, vision, reasoning, structured outputs): query `GET /v1/models`. See [references/models.md](references/models.md).
5. An agent or LLM needs web search or page reading: Clonvo's web tools, over MCP if the framework speaks it, otherwise REST. See [references/web-tools.md](references/web-tools.md).
6. Vectors for search or RAG: `/v1/embeddings` with an embedding model. See [references/embeddings.md](references/embeddings.md).
7. Moving an app from OpenAI or OpenRouter: see the `clonvo-migrate` skill or [references/sdks-and-frameworks.md](references/sdks-and-frameworks.md).
8. A request failed or cost more than expected: read the error code, then look the request up. See [references/errors.md](references/errors.md).

## Critical pitfalls

- Don't guess model ids or add version suffixes from memory. Copy them from `GET /v1/models`. An unknown id returns `400 model_not_found`, and one unknown id in `models` fails the whole request.
- Clonvo extensions (`models`, `provider`, `reasoning`, `top_k`) go in `extra_body` with the Python SDK. In TypeScript they're sent as extra properties; add `// @ts-expect-error Clonvo extension` above them.
- `provider.order` accepts only `"byok"` and `"credits"`. It chooses between the user's own provider keys and Clonvo credits, not between upstream providers like OpenRouter's does. Other entries are ignored.
- `models` and `provider` need Pay as you go or above. On Free they return `402 plan_feature_unavailable`, not a silent no-op.
- Fallbacks happen only before the first streamed chunk, and never for `400` errors (an unsupported parameter, or a prompt longer than the model's context).
- The final streaming chunk has an empty `choices` array and carries `usage`. Check `choices` before indexing it. A stream that fails mid-way ends with `finish_reason: "error"` and an `error` object, with HTTP status already `200`.
- `n` must be `1`. Send separate requests for several completions.
- The Responses API is stateless. `previous_response_id`, `conversation` and `background` return `400 unsupported_parameter`. Resend the earlier input plus the previous `output` items every turn.
- Files go inline as base64 `file_data`. There's no upload API, and `file_id` returns `400`.
- Sending a content part a model can't take (an image to a text-only model, a PDF to a provider without file input) returns `400 unsupported_content`. Check the model's `architecture.input_modalities` first.
- `tool_choice` values a model can't honour become `auto` instead of failing. Check the response for `tool_calls` rather than assuming one was made.
- Don't retry `400`, `401`, `402`, `403` or `404`; the same request fails again. Retry `429` and `5xx` with backoff and respect `Retry-After`.
- Free models (`:free`) need an owner or admin to turn them on in **Settings → Privacy** (otherwise `403 training_providers_not_allowed`). Their prompts may be stored and used for training, so never send confidential data to them.

More: [references/common-mistakes.md](references/common-mistakes.md).

## Reference files

| File | Topics |
| --- | --- |
| [references/chat-completions.md](references/chat-completions.md) | Parameters, extensions, response fields, streaming, tool calling, structured outputs, reasoning, images, PDFs and audio, prompt caching |
| [references/responses-api.md](references/responses-api.md) | Stateless Responses API, conversation replay, function calling, OpenAI Agents SDK |
| [references/models.md](references/models.md) | Model id format, aliases, free models, `GET /v1/models` fields, choosing by capability and price, peak-hour prices |
| [references/routing-and-billing.md](references/routing-and-billing.md) | Fallback models, own provider keys, `provider.order`, data collection, cost, credits, spending limits, plans |
| [references/web-tools.md](references/web-tools.md) | `web_search` and `web_scrape` over REST and MCP, arguments, giving them to a model, provider routing, tool errors |
| [references/embeddings.md](references/embeddings.md) | Embedding models, request and response, dimensions, framework notes |
| [references/sdks-and-frameworks.md](references/sdks-and-frameworks.md) | OpenAI Python and Node SDKs, environment variables, Vercel AI SDK, LangChain, OpenRouter-style clients, app attribution |
| [references/errors.md](references/errors.md) | Error shape, every status and code, retry policy, looking up a request, checking a key and credits |
| [references/common-mistakes.md](references/common-mistakes.md) | Wrong-to-right corrections for the mistakes agents make most |

## Canonical docs

Every docs page is Markdown at its URL plus `.md`. When this skill and the docs disagree, the docs win.

- Index for agents: https://clonvo.com/llms.txt
- Everything in one file: https://clonvo.com/llms-full.txt
- Quickstart: https://clonvo.com/docs/quickstart.md
- Chat completions reference: https://clonvo.com/docs/chat-completions.md
- Model catalogue: https://clonvo.com/models.md
- Pricing and plans: https://clonvo.com/pricing.md
