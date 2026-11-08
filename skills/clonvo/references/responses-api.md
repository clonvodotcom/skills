# Responses API

`POST https://api.clonvo.com/v1/responses` implements OpenAI's Responses API for every model in the catalogue. Use it when code already uses `client.responses.*` or the OpenAI Agents SDK; otherwise prefer Chat Completions.

Canonical docs: https://clonvo.com/docs/responses.md

## Stateless

Clonvo doesn't store responses. Every response reports `"store": false`, whatever you send.

- To continue a conversation, send all of it in `input`: the earlier input items, then the previous response's `output` items, then the new message or `function_call_output` items.
- `previous_response_id`, `conversation`, `background` and `prompt` return `400 unsupported_parameter`.
- `GET`/`DELETE /v1/responses/{id}` and the other `/v1/responses/…` endpoints return `400 responses_not_stored`.
- `include`, `truncation`, `service_tier`, `prompt_cache_key`, `safety_identifier`, `top_logprobs`, `max_tool_calls` and `stream_options` are accepted and ignored. Encrypted reasoning isn't returned.

## Basic call

```python
response = client.responses.create(
    model="anthropic/claude-sonnet-5-5",
    instructions="Answer in one sentence.",
    input="Why is the sky blue?",
)
print(response.output_text)  # computed by the SDK from the message items
```

## Function calling

```python
input = [{"role": "user", "content": "What's the weather in Paris?"}]
response = client.responses.create(model="google/gemini-3.8-flash", input=input, tools=tools)

input += response.output  # the WHOLE output, not just the function calls
for item in response.output:
    if item.type == "function_call":
        args = json.loads(item.arguments)
        input.append({"type": "function_call_output", "call_id": item.call_id, "output": get_weather(**args)})

response = client.responses.create(model="google/gemini-3.8-flash", input=input, tools=tools)
```

- Tools are flat: `{"type": "function", "name": …, "description": …, "parameters": {…}, "strict": …}`.
- `strict` is sent only when you set it. OpenAI defaults it on in this API; Clonvo doesn't, because most providers can't enforce it.
- Built-in tools (`web_search`, `file_search`, `code_interpreter`, `image_generation`, computer use, `mcp`, shell, `apply_patch`, custom tools) return `400 unsupported_tool`. For web search, offer Clonvo's `web_search`/`web_scrape` as function tools (schemas from `GET /v1/tools`) and run the calls through `POST /v1/tools/{id}`, or connect the MCP server from your own process. See web-tools.md.
- Send reasoning items back with the rest of the output. OpenAI models produce no `reasoning` items here (they run through Chat Completions upstream), but `usage.output_tokens_details.reasoning_tokens` still reports them.

## Input items

- `message` with `role` `user`, `assistant`, `system` or `developer`. User parts: `input_text`, `input_image` (https URL or base64 data URL), `input_file` (base64 `file_data`). Assistant parts: `output_text`.
- `file_id` and `file_url` aren't supported (Clonvo doesn't store files). `item_reference` items return `400`, except references to reasoning items (`rs_…`), which are skipped.

## OpenAI Agents SDK

```python
from openai import AsyncOpenAI
from agents import Agent, OpenAIResponsesModel, Runner, set_tracing_disabled

client = AsyncOpenAI(base_url="https://api.clonvo.com/v1", api_key=os.environ["CLONVO_API_KEY"])
set_tracing_disabled(True)  # otherwise traces go to OpenAI with your Clonvo key

agent = Agent(
    name="Assistant",
    instructions="…",
    tools=[my_tool],
    model=OpenAIResponsesModel(model="anthropic/claude-sonnet-5-5", openai_client=client),
)
result = Runner.run_sync(agent, "…")
```

```ts
import OpenAI from "openai";
import { Agent, OpenAIResponsesModel, run, setTracingDisabled } from "@openai/agents";

const client = new OpenAI({ baseURL: "https://api.clonvo.com/v1", apiKey: process.env.CLONVO_API_KEY });
setTracingDisabled(true);

const agent = new Agent({ name: "Assistant", instructions: "…", tools: [myTool], model: new OpenAIResponsesModel(client, "anthropic/claude-sonnet-5-5") });
const result = await run(agent, "…");
```

- Pass the model explicitly, as above, so any catalogue model can run the agent.
- Turn tracing off, or give the SDK a separate OpenAI key for tracing.
- Don't use server-managed conversations (`previous_response_id`, `conversation_id`, OpenAI conversation sessions). The SDK's default of resending the conversation works.
- Hosted tools such as `WebSearchTool()` aren't supported. Use Clonvo's MCP server with `MCPServerStreamableHttp` for web tools (see web-tools.md).
