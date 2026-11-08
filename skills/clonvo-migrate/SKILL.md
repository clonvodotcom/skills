---
name: clonvo-migrate
description: "Move an app or agent from the OpenAI API or OpenRouter to Clonvo: find the call sites, swap base URL and key, map model ids, replace features Clonvo doesn't have, translate OpenRouter routing options, and verify with a smoke test. Use when the user wants to switch to Clonvo, use Clonvo instead of OpenAI or OpenRouter, or point existing OpenAI-SDK code at Clonvo."
metadata:
  author: Clonvo
  version: "1.0.0"
  docs: "https://clonvo.com/docs/openai-sdks.md"
---

# Migrate to Clonvo

Clonvo speaks OpenAI's API and accepts OpenRouter's model ids, so most migrations are a base URL, a key and model ids. The work is in the few features that differ. Follow the steps in order and report what you changed and what you couldn't.

## 1. Find the call sites

Search the codebase for:

- Base URLs: `api.openai.com`, `openrouter.ai/api`, `baseURL`, `base_url`, `OPENAI_BASE_URL`
- Keys: `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, `sk-or-`
- Clients: `new OpenAI(`, `OpenAI(`, `AsyncOpenAI(`, `createOpenAI(`, `ChatOpenAI(`, `OpenAIEmbeddings(`, `@openrouter/`
- Model strings: `"gpt-`, `"o1`, `"o3`, `"o4`, `"claude-`, `"gemini-`, and any `author/slug` ids
- Endpoints Clonvo doesn't have (step 4): `images.`, `audio.`, `files.`, `batches.`, `beta.assistants`, `beta.threads`, `vector_stores`

List them before editing so nothing is missed.

## 2. Swap base URL and key

| From | To |
| --- | --- |
| `https://api.openai.com/v1` | `https://api.clonvo.com/v1` |
| `https://openrouter.ai/api/v1` | `https://api.clonvo.com/v1` (or `https://clonvo.com/api/v1`, same API) |
| `OPENAI_API_KEY` / `OPENROUTER_API_KEY` | `CLONVO_API_KEY` (`sk-cv-v1-…`) |

```python
client = OpenAI(base_url="https://api.clonvo.com/v1", api_key=os.environ["CLONVO_API_KEY"])
```

For tools you can't edit, the OpenAI SDKs read `OPENAI_BASE_URL=https://api.clonvo.com/v1` and `OPENAI_API_KEY=$CLONVO_API_KEY`.

Keep the key in the environment or secret store, and add `CLONVO_API_KEY` to `.env.example` or the deployment config, not to code.

## 3. Map model ids

Fetch the catalogue once: `curl -s https://api.clonvo.com/v1/models -o /tmp/clonvo-models.json`.

- **From OpenAI**: bare ids become `openai/<slug>`. Check each exists: `jq -r '.data[].id | select(startswith("openai/"))' /tmp/clonvo-models.json`.
- **From OpenRouter**: ids for OpenAI, Anthropic, Google, xAI, DeepSeek, Mistral, Moonshot and Meta usually work unchanged. Check each one; Clonvo's catalogue covers those eight providers (plus `:free` models), so ids from other authors need a replacement. Ask the user which model to use instead; don't pick silently.
- **Date-stamped or preview ids** (`…-20260115`, `…-preview`): keep them only if that exact id is in the catalogue (some dated ids are); otherwise use the current catalogue id and tell the user.
- `:free` ids work only from OpenRouter's current free list, and only after an owner or admin enables free models in **Settings → Privacy**.

Use the `clonvo-models` skill to resolve names and compare prices.

## 4. Replace what Clonvo doesn't have

| Feature | On Clonvo |
| --- | --- |
| Images, Audio (speech, transcription), Files, Batch, Assistants/Threads, Vector stores | Not available: `404 unknown_endpoint`. Keep those calls on the original provider, and tell the user. |
| `file_id` in messages | Send files inline as base64 `file_data` |
| `n > 1` | Send separate requests |
| Responses API state (`previous_response_id`, `conversation`, `store`, `background`) | Stateless. Resend earlier input plus the previous `output` items each turn |
| Responses built-in tools (`web_search`, `file_search`, `code_interpreter`, `image_generation`, `mcp`) | `400 unsupported_tool`. For web search use Clonvo's `web_search`/`web_scrape` (REST or MCP); others have no equivalent |
| OpenAI Agents SDK tracing | Disable it (`set_tracing_disabled(True)`); it would send your Clonvo key to OpenAI |
| Anthropic SDK (`/v1/messages`) | Not served. Move those calls to the OpenAI SDK with `anthropic/…` ids |

## 5. Translate OpenRouter options

| OpenRouter | Clonvo |
| --- | --- |
| `models: [...]` fallbacks | Same (up to 5). Needs Pay as you go or above |
| `provider.order` with upstream providers (`"anthropic"`, `"together"`) | Clonvo's `provider.order` takes only `"byok"` and `"credits"`. Remove upstream provider names; choose models with `models` instead |
| `provider.only`, `ignore`, `sort`, `quantizations`, `max_price`, `require_parameters` | Not part of Clonvo's chat API. Remove them |
| `provider.data_collection: "deny"` | Same |
| `provider.allow_fallbacks` | Same name; applies to `provider.order` only |
| `reasoning: {effort, max_tokens, exclude, enabled}` | Same |
| `usage: {include: true}` | Accepted; usage is always included, with `usage.cost` |
| `HTTP-Referer`, `X-Title` headers | Same; they name the app in Clonvo's activity |
| `:online` suffix or the web plugin | Not part of Clonvo's API. Use Clonvo's web tools |
| `GET /api/v1/models`, `/key`, `/credits`, `/generation` | Same formats at `https://api.clonvo.com/v1/…` (no provisioning keys) |
| BYOK | Add provider keys in Clonvo's dashboard under **Provider keys**; first $25,000/month of own-key usage has no fee |

## 6. Check the plan

On the Free plan only `:free` models work, and `models`/`provider` return `402 plan_feature_unavailable`. If the code uses paid models or fallbacks, the organization needs credits (any purchase moves it to Pay as you go).

## 7. Verify

Run one real request per distinct model and feature (streaming, tools, structured output), then read the result:

```bash
curl -s https://api.clonvo.com/v1/chat/completions \
  -H "Authorization: Bearer $CLONVO_API_KEY" -H "Content-Type: application/json" \
  -d '{"model": "openai/gpt-6.1-sol", "messages": [{"role": "user", "content": "Say ok."}], "max_completion_tokens": 20}' \
  | jq '{model, provider, content: .choices[0].message.content, cost: .usage.cost, error}'
```

Swap in a model id from step 3. Then run the project's own tests. Report:

- the files changed and the model mapping used,
- features left on the original provider (step 4),
- anything not verified (for example, no key available to run the smoke test).

Use the `clonvo` skill for details on any feature.
