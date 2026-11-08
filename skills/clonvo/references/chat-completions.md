# Chat Completions

`POST https://api.clonvo.com/v1/chat/completions` follows OpenAI's Chat Completions API. Fields marked **extension** are Clonvo or OpenRouter additions: pass them with `extra_body` in Python, or as extra properties in Node.js (with `// @ts-expect-error` in TypeScript).

Canonical docs: https://clonvo.com/docs/chat-completions.md

## Request parameters

| Parameter | Notes |
| --- | --- |
| `model` | Required. Canonical id such as `anthropic/claude-sonnet-5-5`. |
| `messages` | Required. Roles: `system`, `developer` (mapped to `system` where needed), `user`, `assistant`, `tool`. |
| `max_completion_tokens` / `max_tokens` | Includes reasoning tokens. If omitted and the balance can't cover the model's maximum output, Clonvo caps the response at what credits can pay for (it ends with `finish_reason: "length"`). A value you set is never lowered. |
| `temperature`, `top_p`, `stop`, `seed`, `presence_penalty`, `frequency_penalty`, `logit_bias`, `logprobs`, `top_logprobs` | Passed where the provider supports them; unsupported sampling parameters are dropped silently. Out-of-range values (e.g. `temperature` above 2) return `400`. |
| `top_k` | Extension. Sent to Anthropic (reasoning off), Gemini and free models. |
| `n` | Only `1`. Anything else returns `400`. |
| `verbosity` | `low`, `medium`, `high`; OpenAI models that accept it. |
| `tools`, `tool_choice`, `parallel_tool_calls` | See Tool calling below. |
| `response_format` | `text`, `json_object`, or `json_schema`. See Structured outputs below. |
| `reasoning_effort` | `none`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`. |
| `reasoning` | Extension (OpenRouter style): `effort`, `max_tokens`, `exclude`, `enabled`. Wins over `reasoning_effort`. |
| `stream` | Server-sent events. `stream_options` is accepted; the usage chunk is always sent. |
| `models` | Extension. Up to 5 fallback models. See routing-and-billing.md. |
| `provider` | Extension. `order` (`"byok"`/`"credits"`), `allow_fallbacks`, `data_collection` (`"allow"`/`"deny"`). |
| `user` | Passed to OpenAI, xAI, Anthropic and free models; not stored. |
| `metadata`, `service_tier`, `usage.include` | Accepted for compatibility; no effect. |

Unknown top-level fields are ignored. Send `HTTP-Referer` and `X-Title` headers to name your app in activity and `GET /v1/generation`.

## Response

```json
{
  "id": "gen-01M67FGKKVQ8ZK2M3N5P7R9S1V",
  "object": "chat.completion",
  "model": "anthropic/claude-sonnet-5-5",
  "provider": "Anthropic",
  "choices": [{ "index": 0, "message": { "role": "assistant", "content": "…" }, "finish_reason": "stop" }],
  "usage": {
    "prompt_tokens": 1200,
    "completion_tokens": 310,
    "total_tokens": 1510,
    "prompt_tokens_details": { "cached_tokens": 1024, "cache_write_tokens": 0 },
    "completion_tokens_details": { "reasoning_tokens": 0 },
    "cost": 0.0035544,
    "is_byok": false
  }
}
```

- `finish_reason`: `stop`, `length`, `tool_calls`, `content_filter` or `error`.
- `message.reasoning` (extension): reasoning text, when the model returns it.
- `usage.cost` is the list-price cost in USD. On credits it's exactly what was deducted; on the user's own key it's what the request would have cost on credits.
- Python: read extensions with `completion.usage.cost` (attribute access works) or `completion.model_extra["provider"]`. For the `x-clonvo-generation-id` header, use `client.chat.completions.with_raw_response.create(…)`.

## Streaming

```python
stream = client.chat.completions.create(model="google/gemini-3.8-flash", messages=messages, stream=True)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
    if chunk.usage:
        print("\ncost:", chunk.usage.cost)
```

- Every stream ends with a chunk whose `choices` is empty and which carries `usage`, then `data: [DONE]`. Always check `choices` before indexing.
- `delta.reasoning` (extension) streams reasoning text; `delta.tool_calls` streams tool call fragments.
- SSE comments (`: keep-alive`) arrive every ~15 s of silence. SDKs ignore them; skip lines starting with `:` in a hand-written parser.
- Clonvo holds the response for up to 3 s waiting for the provider's first content, so a failing provider can still be replaced by a fallback.
- Errors before the stream starts are normal JSON errors. Errors after it starts arrive as a chunk with an `error` object (`message`, `type`, `code`, `status`) and `finish_reason: "error"`, then the usage chunk. Treat that as a failed generation; the tokens produced are charged.
- Closing the connection cancels the provider request; tokens produced so far are charged.

Docs: https://clonvo.com/docs/streaming.md

## Tool calling

OpenAI's format for every provider; Clonvo translates for Anthropic and Gemini.

```python
import json

completion = client.chat.completions.create(model="anthropic/claude-sonnet-5-5", messages=messages, tools=tools)
message = completion.choices[0].message
messages.append(message)
for call in message.tool_calls or []:
    result = run_tool(call.function.name, json.loads(call.function.arguments))
    messages.append({"role": "tool", "tool_call_id": call.id, "content": json.dumps(result)})
# Call again with the same tools; repeat until finish_reason is "stop".
```

- `tool_choice`: `auto` (default), `none`, `required`, or `{"type": "function", "function": {"name": "…"}}`. A value the model can't honour becomes `auto`, so check for `tool_calls`. On Anthropic, forcing a tool turns extended thinking off for that request.
- Rejected with `400` before reaching a provider: `required` or a named function with no `tools`; a named function not in `tools`; duplicate tool names; `tools` sent to a model without tool support (`unsupported_parameter`).
- `parallel_tool_calls: false` is honoured by OpenAI, xAI, Mistral, Anthropic and free models; Gemini, DeepSeek and Moonshot ignore it.
- `"strict": true` inside `function` is enforced only by providers that support it. Validate arguments anyway.
- Streaming: fragments arrive in `delta.tool_calls` by `index`. The first fragment carries `id` and `function.name`; concatenate `function.arguments`.
- Switching models mid-conversation is fine: keep the same history and Clonvo converts tool calls and results for the new provider.
- Find models with tool support: `supported_parameters` contains `tools` in `GET /v1/models`.

Docs: https://clonvo.com/docs/tool-calling.md

## Structured outputs

```python
completion = client.chat.completions.create(
    model="openai/gpt-6.1-sol",
    messages=messages,
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "event",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {"title": {"type": "string"}, "attendees": {"type": "integer"}},
                "required": ["title", "attendees"],
                "additionalProperties": False,
            },
        },
    },
)
event = json.loads(completion.choices[0].message.content)
```

- `client.beta.chat.completions.parse(response_format=MyPydanticModel)` works through Clonvo.
- `response_format` never causes a `400` because of the model. Models without native schema support get a best-effort fallback (JSON mode plus the schema as an instruction, or a forced tool on older Anthropic models), so validate the result.
- `json_object` guarantees valid JSON, not a shape: describe the shape in the prompt.
- Keep strict schemas simple: every property required, `additionalProperties: false`.
- If `finish_reason` is `length`, the JSON was cut off; raise `max_completion_tokens`.
- Models with native schema support list `structured_outputs` in `supported_parameters`.

Docs: https://clonvo.com/docs/structured-outputs.md

## Reasoning

```python
client.chat.completions.create(model="anthropic/claude-opus-5-5", messages=messages, reasoning_effort="high")
client.chat.completions.create(model="deepseek/deepseek-v4-pro", messages=messages, extra_body={"reasoning": {"effort": "medium"}})
```

- Each model accepts a subset of efforts. An unsupported value isn't an error: Clonvo uses the next higher supported effort (or the model's highest). `none` is never raised.
- `reasoning.exclude: true` reasons without returning the text; `reasoning.enabled: false` turns it off; `reasoning.max_tokens` sets a budget on models that take one.
- Without a reasoning parameter each model uses its provider default; DeepSeek and Kimi only reason when asked.
- Reasoning tokens are output tokens: in `completion_tokens`, broken out in `completion_tokens_details.reasoning_tokens`, charged at the output price. Leave room for them in `max_completion_tokens`.
- In tool loops, pass assistant messages back as received, including `reasoning`. Clonvo restores Anthropic thinking blocks and Gemini thought signatures for 24 hours, keyed by tool call id. DeepSeek and Kimi get `reasoning` back as `reasoning_content`.

Docs: https://clonvo.com/docs/reasoning.md

## Images, PDFs and audio

User `content` can be an array of parts:

```python
{"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg", "detail": "high"}}
{"type": "file", "file": {"filename": "contract.pdf", "file_data": f"data:application/pdf;base64,{b64}"}}
{"type": "input_audio", "input_audio": {"data": "<base64>", "format": "wav"}}
```

- Images: public `https` URL or base64 data URL. Clonvo fetches URLs for providers that can't (Gemini), up to 20 MB per image and 50 MB per request (`400 media_too_large` beyond that).
- Files: inline base64 only; `file_id` returns `400`. Accepted by OpenAI, Anthropic (PDFs) and Gemini.
- Audio: OpenAI and Gemini models with audio input.
- A part the model can't take returns `400 unsupported_content`; nothing is dropped silently. Fallback models that can't take the content are skipped.
- Check support in `architecture.input_modalities` (`image`, `file`, `audio`) from `GET /v1/models`.

Docs: https://clonvo.com/docs/images-and-files.md

## Prompt caching

- Cached input is reported in `usage.prompt_tokens_details.cached_tokens` and billed at the model's cache read price.
- For Anthropic models, mark a text part with `cache_control: {"type": "ephemeral"}` (optional `ttl`: `5m` or `1h`) to set a cache breakpoint. Writes are reported in `cache_write_tokens`.
