# Web tools

`web_search` finds pages; `web_scrape` reads one page as Markdown or text. Same Clonvo key as models. Pay as you go and above (`402` on Free); any credit purchase moves an organization there.

The same API and MCP server also have:

| Tool | Does |
| --- | --- |
| `web_map` | List a site's URLs |
| `web_extract` | Typed JSON from one page, shaped by your JSON Schema |
| `web_answer` | Short answer from current sources, with citations |
| `find_similar` | Pages similar to a URL |
| `company_search`, `company_enrich`, `company_news`, `company_jobs`, `company_similar` | Company data |
| `people_search`, `people_enrich` | Professional profiles |
| `web_crawl`, `web_research`, `browser_task` | Long-running: crawl a site, a cited research report, a browser agent working toward a goal. They run as jobs |

`GET /v1/tools` lists every tool with its `input_schema`. Jobs: REST returns a job to poll at `GET /v1/tools/jobs/{id}`; over MCP, clients that support tasks get one, others get a job id for the free `job_status` tool.

Calls run on the organization's own Firecrawl, Tavily, Exa, Parallel or TinyFish key when one is added under **Provider keys → Tools**; otherwise on Clonvo credits through treg (the call's cost plus 2%). Own keys are always tried first.

Canonical docs: https://clonvo.com/docs/tools.md and https://clonvo.com/docs/mcp.md

## Arguments

`web_search`:

| Argument | Notes |
| --- | --- |
| `query` | Required. A question or keywords. |
| `max_results` | 1–20, default 5. |
| `include_domains` / `exclude_domains` | Hard filters. Use only when the user names the sites. |
| `time_range` | `day`, `week`, `month`, `year`. |
| `include_content` | Also return each page's content (up to `max_content_chars`, default 5,000). Slower and costs more on some providers. |

`web_scrape`:

| Argument | Notes |
| --- | --- |
| `url` | Required. http(s). |
| `format` | `markdown` (default) or `text`. |
| `only_main_content` | Default `true`. Some providers always extract main content. |
| `max_chars` | Default 50,000; longer pages are cut and `truncated` is `true`. |

A provider that can't honour an argument exactly still serves the call and explains in `result.notes`.

## REST

```bash
curl https://api.clonvo.com/v1/tools/web_search \
  -H "Authorization: Bearer $CLONVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"arguments": {"query": "PostgreSQL 18 release notes", "max_results": 3}}'
```

```json
{
  "id": "gen-01M69T8V4J2QH3K5N7P9R1S3TX",
  "tool": "web_search",
  "provider": "tavily",
  "result": {
    "query": "PostgreSQL 18 release notes",
    "results": [{ "title": "PostgreSQL 18 Release Notes", "url": "https://www.postgresql.org/docs/18/release-18.html", "snippet": "…", "published_at": null, "score": 0.86 }]
  },
  "usage": { "cost": 0.008, "charged": 0, "is_byok": true }
}
```

- The body is `{"arguments": {…}}`, optionally with `"provider": {…}`. Arguments at the top level are rejected.
- `web_scrape` returns `result.url`, `title`, `content`, `format`, `truncated`, `status_code`.
- `usage.cost` is the list price; `usage.charged` is what came out of credits (`0` on own keys within the BYOK allowance). `provider: "treg"` means Clonvo credits, and `served_by` names the provider treg used.
- `GET /v1/tools` lists each tool with `input_schema` and `available_providers`. An empty `available_providers` means the tool can't be called yet (no tool key and no credits).

## Giving the tools to a model

`input_schema` from `GET /v1/tools` is a valid function `parameters` object. Offer the tools through tool calling and run each call through `POST /v1/tools/{id}`:

```python
import json, requests

tool_defs = requests.get("https://api.clonvo.com/v1/tools", headers=auth, timeout=30).json()["data"]
tools = [{"type": "function", "function": {"name": t["id"], "description": t["description"], "parameters": t["input_schema"]}} for t in tool_defs]

completion = client.chat.completions.create(model="anthropic/claude-sonnet-5-5", messages=messages, tools=tools)
message = completion.choices[0].message
messages.append(message)
for call in message.tool_calls or []:
    resp = requests.post(
        f"https://api.clonvo.com/v1/tools/{call.function.name}",
        headers=auth,
        json={"arguments": json.loads(call.function.arguments)},
        timeout=60,
    )
    body = resp.json()
    content = body["result"] if resp.ok else {"error": body["error"]["message"]}
    messages.append({"role": "tool", "tool_call_id": call.id, "content": json.dumps(content)})
# Call the model again; repeat until finish_reason is "stop".
```

Return tool errors to the model as the tool result (as above) so it can adapt, rather than raising.

## MCP

Remote server, Streamable HTTP, stateless. Apps sign in with OAuth (no key to paste); agents and scripts send an API key. Setup for every client: https://github.com/clonvodotcom/mcp. Registry name: `com.clonvo/mcp`.

| Setting | Value |
| --- | --- |
| URL | `https://api.clonvo.com/mcp` |
| Auth | OAuth sign-in, or `Authorization: Bearer <CLONVO_API_KEY>` |

ChatGPT, Claude and other apps with custom connectors: add the URL as a connector and sign in.

Claude Code, signing in (then `/mcp` → **Authenticate**):

```bash
claude mcp add --transport http clonvo https://api.clonvo.com/mcp
```

With an API key:

```bash
claude mcp add --transport http clonvo https://api.clonvo.com/mcp \
  --header "Authorization: Bearer $CLONVO_API_KEY"
```

Project `.mcp.json` with a key (each person sets `CLONVO_API_KEY` themselves):

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

Cursor (`.cursor/mcp.json`): `{"mcpServers": {"clonvo": {"url": "https://api.clonvo.com/mcp"}}}` signs in; add `"headers"` with `"Bearer ${env:CLONVO_API_KEY}"` to use a key.

Codex: `codex mcp add clonvo --url https://api.clonvo.com/mcp`, then `codex mcp login clonvo`; or add with `--bearer-token-env-var CLONVO_API_KEY`.

OpenAI Agents SDK (Python), keeping the key in your process:

```python
from agents.mcp import MCPServerStreamableHttp

async with MCPServerStreamableHttp(
    name="Clonvo",
    params={"url": "https://api.clonvo.com/mcp", "headers": {"Authorization": f"Bearer {os.environ['CLONVO_API_KEY']}"}, "timeout": 60},
    cache_tools_list=True,
) as server:
    agent = Agent(name="Researcher", instructions="Use web_search to find sources and web_scrape to read them. Cite URLs.", mcp_servers=[server])
```

Vercel AI SDK:

```ts
import { createMCPClient } from "@ai-sdk/mcp";

const clonvo = await createMCPClient({
  transport: { type: "http", url: "https://api.clonvo.com/mcp", headers: { Authorization: `Bearer ${process.env.CLONVO_API_KEY}` } },
});
const tools = await clonvo.tools(); // pass to generateText / streamText; close() when done
```

- A tool problem (bad arguments, no provider, plan, provider failure) comes back as a tool result with `isError: true`, readable by the agent. A bad or missing Clonvo key or OAuth token is an HTTP `401`.
- Clients that only support the old HTTP+SSE transport, or only local servers, can't connect.

## Provider routing

`"provider": {"order": [...], "only": [...], "ignore": [...], "allow_fallbacks": true}` with `tavily`, `exa`, `firecrawl`, `parallel`, `tinyfish` and `treg` (Clonvo credits). `{"ignore": ["treg"]}` keeps calls on the user's own keys. Over MCP, put it in the call's `_meta["com.clonvo/provider"]` (most clients can't).

## Tool errors

| Status | Code | Fix |
| --- | --- | --- |
| `400` | `invalid_tool_arguments` | `param` names the bad argument, e.g. `arguments.max_results`. |
| `400` | `tool_requires_own_key` | No usable provider after the `provider` settings, and credits excluded. Add a tool key or loosen `only`/`ignore`. |
| `400` | `no_allowed_providers` | `provider` settings exclude every provider. |
| `402` | `byok_requires_paid_plan`, `insufficient_credits`, `key_limit_exceeded`, `byok_fee_requires_credits` | Plan or credits. Read the message. |
| `404` | `tool_not_found` | Use an id from `GET /v1/tools`. |
| `429` | `tool_spend_limit_reached` | Try later or add an own key. |
| `502` | `route_failed`, `tool_price_limit` | Retry shortly. |
| `503` | `tools_unavailable` | Retry in a few minutes or add an own key. |
