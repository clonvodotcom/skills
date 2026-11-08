---
name: clonvo-web-tools
description: "Search the web and read web pages through Clonvo's web_search and web_scrape tools, using the Clonvo MCP server's tools when connected or Clonvo's REST API with CLONVO_API_KEY otherwise. Use when a task needs current information, sources, facts past your training data, or the contents of a URL, and the user has a Clonvo API key or the Clonvo MCP server connected."
metadata:
  author: Clonvo
  version: "1.0.0"
  docs: "https://clonvo.com/docs/tools.md"
---

# Clonvo web tools

Two tools: `web_search` (find pages) and `web_scrape` (read one page). They run on the user's own Firecrawl, Tavily, Exa, Parallel or TinyFish key if their organization added one, otherwise on Clonvo credits. They need Pay as you go or above.

## Which way to call them

1. If your tool list has `web_search` and `web_scrape` from the Clonvo MCP server (often prefixed, such as `mcp__clonvo__web_search`), use them.
2. Otherwise, if `CLONVO_API_KEY` is set, call the REST API with `curl` (below).
3. Otherwise, tell the user how to connect, and stop:

```bash
export CLONVO_API_KEY="sk-cv-v1-…"   # from the Clonvo dashboard, API keys
claude mcp add --transport http clonvo https://api.clonvo.com/mcp --header "Authorization: Bearer $CLONVO_API_KEY"
```

Never print the key or put it in a URL.

| You have | You need | Use |
| --- | --- | --- |
| A question or topic | Sources, facts, news | `web_search` |
| A URL | Its content | `web_scrape` |
| A question needing full pages | Answer with evidence | `web_search`, then `web_scrape` the best 1–3 results |

## web_search

```bash
curl -s https://api.clonvo.com/v1/tools/web_search \
  -H "Authorization: Bearer $CLONVO_API_KEY" -H "Content-Type: application/json" \
  -d '{"arguments": {"query": "PostgreSQL 18 release date", "max_results": 8}}'
```

Arguments: `query` (required), `max_results` (1–20, default 5), `time_range` (`day`, `week`, `month`, `year`), `include_domains`, `exclude_domains`, `include_content`, `max_content_chars`. Results are in `result.results[]` with `title`, `url`, `snippet` (≤500 characters), and `published_at` and `score` when known.

- **Write queries for a search engine.** Use the words you expect on the page, not the user's whole request. Split a multi-part question into one focused search per part.
- **Batch independent searches** in one turn, one call each.
- **Ask for 8–10 results** when the answer isn't obvious; the default of 5 is for simple lookups.
- **Use `time_range`** for news and recent events. Put recency in the query too ("2026", "latest").
- **Use `include_domains` only for a hard allowlist** the user asked for or an official source you know holds the answer. It's a hard filter: search again without it if nothing comes back.
- **Don't set `include_content` by default.** Read the snippets, then `web_scrape` the pages worth reading.
- **Follow leads.** When a result names a likely answer, confirm it with a second search or by reading the page.

## web_scrape

```bash
curl -s https://api.clonvo.com/v1/tools/web_scrape \
  -H "Authorization: Bearer $CLONVO_API_KEY" -H "Content-Type: application/json" \
  -d '{"arguments": {"url": "https://www.postgresql.org/about/"}}'
```

Arguments: `url` (required), `format` (`markdown` default, or `text`), `only_main_content` (default `true`), `max_chars` (default 50,000). The page is in `result.content`; `result.truncated` is `true` when it was cut.

- **Keep large pages out of your context.** With `curl`, save the response to a file (`-o page.json`) and read the parts you need. Lower `max_chars` when you only need the top of a page.
- If a page comes back empty or blocked, try another result rather than retrying the same URL.

## Reading results

- Count independent sources, not results. Prefer primary sources (official docs, filings, the organization's own site) over aggregators.
- Cite the URLs you used.
- `result.notes` explains when a provider handled an argument differently (for example, no relevance scores).

## Errors

| Response | Meaning | Do |
| --- | --- | --- |
| `401` | Bad or missing key | Ask the user to check `CLONVO_API_KEY` |
| `402` | Free plan, no credits, or the key's spending limit | Tell the user; don't retry |
| `400 invalid_tool_arguments` | `param` names the bad argument | Fix it and retry once |
| `400 tool_requires_own_key` | No usable provider | Tell the user to add a tool key or credits |
| `429`, `502`, `503` | Busy or unavailable | Wait briefly, retry once |

Over MCP, these come back as a tool result with `isError: true` and the same message.

## Cost

Each call reports `usage.cost` (list price) and `usage.charged` (taken from credits; `0` on the user's own key within the allowance). Don't loop searches without new information to look for. Full details: https://clonvo.com/docs/tools.md
