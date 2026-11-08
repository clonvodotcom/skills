# SDKs and frameworks

Anything that lets you change the OpenAI base URL works with Clonvo. Use `https://api.clonvo.com/v1`, the Clonvo key, and Clonvo model ids.

Canonical docs: https://clonvo.com/docs/openai-sdks.md

## OpenAI Python SDK

```python
client = OpenAI(base_url="https://api.clonvo.com/v1", api_key=os.environ["CLONVO_API_KEY"])

client.chat.completions.create(
    model="anthropic/claude-sonnet-5-5",
    messages=messages,
    extra_body={"models": ["openai/gpt-6.1-sol"], "provider": {"data_collection": "deny"}},
)
```

- Extensions in the request: `extra_body`.
- Extensions in the response: `completion.usage.cost`, `completion.model_extra["provider"]`.
- Generation id header: `client.chat.completions.with_raw_response.create(…)`, then `.headers["x-clonvo-generation-id"]` and `.parse()`.
- The SDK retries `429` and `5xx` a couple of times; tune with `max_retries`.

## OpenAI Node.js SDK

```ts
const client = new OpenAI({ baseURL: "https://api.clonvo.com/v1", apiKey: process.env.CLONVO_API_KEY });
```

Extra request fields are sent as-is. In TypeScript, add `// @ts-expect-error Clonvo extension` or a cast. `client.responses.create(…)` uses the stateless Responses API.

## Environment variables

Many tools built on the OpenAI SDK work with no code change:

```bash
export OPENAI_BASE_URL="https://api.clonvo.com/v1"
export OPENAI_API_KEY="$CLONVO_API_KEY"
```

## Vercel AI SDK

```ts
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";

const clonvo = createOpenAI({ baseURL: "https://api.clonvo.com/v1", apiKey: process.env.CLONVO_API_KEY });

const { text } = await generateText({ model: clonvo.chat("google/gemini-3.8-flash"), prompt: "…" });
```

Use `clonvo.chat(…)`. The provider's default model calls the Responses API, which Clonvo serves statelessly; `.chat(…)` is the tested path.

## LangChain

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="mistralai/mistral-large-4", base_url="https://api.clonvo.com/v1", api_key=os.environ["CLONVO_API_KEY"])
```

For `OpenAIEmbeddings`, set `check_embedding_ctx_length=False`.

## Tools built for OpenRouter

Clonvo accepts OpenRouter-style model ids and also serves the API at `https://clonvo.com/api/v1`, so a tool with an OpenRouter base URL setting usually works by swapping the base URL and key. Behaviour differences are in the `clonvo-migrate` skill.

## Editors and chat apps

For any "OpenAI-compatible" provider option: base URL `https://api.clonvo.com/v1`, the `sk-cv-v1-…` key, and a model id from the catalogue.

## App attribution

Send `HTTP-Referer` and `X-Title` to name the app in the dashboard's activity and in `GET /v1/generation` (`origin`, `app_title`):

```python
client = OpenAI(
    base_url="https://api.clonvo.com/v1",
    api_key=os.environ["CLONVO_API_KEY"],
    default_headers={"HTTP-Referer": "https://example.com", "X-Title": "Example App"},
)
```

## Not available

OpenAI's Images, Audio, Files, Batch and Assistants APIs return `404 unknown_endpoint`. A supported path with the wrong method returns `405 method_not_allowed`.
