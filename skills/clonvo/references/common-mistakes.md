# Common mistakes

Frequent Clonvo integration mistakes, with corrections.

## Setup and ids

| Wrong | Correct |
| --- | --- |
| `base_url="https://api.clonvo.com"` (no `/v1`) | `https://api.clonvo.com/v1` (or `https://clonvo.com/api/v1` for OpenRouter-style clients) |
| Hard-coding `sk-cv-v1-…` in code, or calling Clonvo from a browser or mobile app | Read `CLONVO_API_KEY` from the server environment |
| A model id from memory or a guessed one (`anthropic/claude-sonnet-latest`, `openai/gpt-6`) | An id copied from `GET /v1/models` |
| A bare upstream id (`claude-sonnet-5-5-20261001`, `gemini-3.8-flash-preview`) | The catalogue id (`anthropic/claude-sonnet-5-5`, `google/gemini-3.8-flash`) |
| An alias for a free model (`xai/…:free`) | Free ids exactly as listed; aliases don't apply to `:free` |
| An embedding model in chat completions, or a chat model in `/v1/embeddings` | `400 embedding_model_not_chat` / `model_not_embedding`: use the right endpoint |

## Extensions

| Wrong | Correct |
| --- | --- |
| `client.chat.completions.create(..., models=[...])` in Python | `extra_body={"models": [...]}` |
| `models` in TypeScript without a type escape | `// @ts-expect-error Clonvo extension` above the property |
| `provider: {"order": ["anthropic", "google-vertex"]}` (OpenRouter-style upstream routing) | Clonvo's `provider.order` takes only `"byok"` and `"credits"`; to change models, use `models` |
| `models` or `provider` on the Free plan, expecting them to be ignored | They return `402 plan_feature_unavailable`; buy credits or drop them |
| More than 5 entries in `models` | Up to 5 (`400 too_many_models`) |
| Fallbacks that lack the request's needs (tools, images, structured outputs, context) | Fallbacks with the same capabilities, or they're skipped |
| Expecting a fallback after a `400` | `400`s return straight away; fix the request |

## Requests

| Wrong | Correct |
| --- | --- |
| `n: 3` | `n: 1`; send three requests |
| `previous_response_id` or `store: true` with `/v1/responses` to keep state | Resend earlier input plus the previous `output` items every turn |
| Built-in Responses tools (`{"type": "web_search"}`) | `400 unsupported_tool`. Use Clonvo's `web_search` as a function tool, or the MCP server |
| Uploading a file and sending `file_id` | Inline base64 `file_data` |
| An image or PDF to a model without that input modality | Check `architecture.input_modalities`; otherwise `400 unsupported_content` |
| `tool_choice: "required"` and assuming a tool call came back | Unsupported values become `auto`; check `tool_calls` |
| A tight `max_completion_tokens` on a reasoning model | Reasoning tokens count toward it; leave room |
| `json_object` and expecting a specific shape | Describe the shape in the prompt, or use `json_schema` |
| Trusting `json_schema` output on a model without `structured_outputs` | It's best-effort there; validate |

## Streaming

| Wrong | Correct |
| --- | --- |
| `chunk.choices[0]` on every chunk | The last chunk has empty `choices` and carries `usage`; check first |
| Treating HTTP `200` as success for the whole stream | A mid-stream failure ends with `finish_reason: "error"` and an `error` object |
| Passing `: keep-alive` lines to a JSON parser | Skip SSE comment lines (starting with `:`) |
| Expecting fallbacks after streaming starts | Only before the first chunk |

## Errors and retries

| Wrong | Correct |
| --- | --- |
| Retrying every error | Retry only `429` and `5xx`; respect `Retry-After` |
| Retrying `402` | Credits, the key's limit, or the plan: it fails again until the user acts |
| Looking up `GET /v1/generation` the instant a request ends and treating `404` as missing | Usage lands a few seconds later; retry shortly |
| Free models failing with `403` treated as a bug | An owner or admin must turn them on in **Settings → Privacy** |

## Web tools

| Wrong | Correct |
| --- | --- |
| `POST /v1/tools/web_search` with `{"query": "…"}` | `{"arguments": {"query": "…"}}` |
| `include_domains` added because the user mentioned a site in passing | Only for a hard allowlist the user asked for |
| `include_content: true` on every search | Search first, then `web_scrape` the pages you need, or set it only when you need every page's text |
| Raising tool errors in an agent loop | Return them to the model as the tool result |
| Calling tools on the Free plan | Pay as you go and above (`402`) |
