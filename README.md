# Clonvo agent skills

[![skills.sh](https://skills.sh/b/clonvodotcom/skills)](https://skills.sh/clonvodotcom/skills)

[Agent Skills](https://agentskills.io) for [Clonvo](https://clonvo.com), one OpenAI-compatible API for OpenAI, Anthropic, Google Gemini, xAI Grok, DeepSeek, Mistral, Kimi and Meta models, with fallbacks, your own provider keys, and web search and scraping tools for agents.

They work with any agent that supports skills, including Claude Code, Cursor, Codex, GitHub Copilot, Gemini CLI, Windsurf and OpenCode.

> [!NOTE]
> You need a Clonvo API key. Create one in the [Clonvo dashboard](https://clonvo.com) and set it as `CLONVO_API_KEY` in your agent's environment.

## Skills

| Skill | Use it to |
| --- | --- |
| [`clonvo`](skills/clonvo/SKILL.md) | Build with the API: chat completions, the Responses API, embeddings, streaming, tool calling, structured outputs, reasoning, images and PDFs, fallback models, your own provider keys, web tools, errors and billing. |
| [`clonvo-models`](skills/clonvo-models/SKILL.md) | Find, compare and price models from the live catalogue, and resolve a model name to its exact id. |
| [`clonvo-migrate`](skills/clonvo-migrate/SKILL.md) | Move an app from the OpenAI API or OpenRouter to Clonvo, step by step. |
| [`clonvo-web-tools`](skills/clonvo-web-tools/SKILL.md) | Search the web and read pages with Clonvo's `web_search` and `web_scrape`, over MCP or REST. |

## Install

All skills, with the [skills CLI](https://skills.sh):

```bash
npx skills add clonvodotcom/skills
```

One skill:

```bash
npx skills add clonvodotcom/skills --skill clonvo
```

### Claude Code plugin

The plugin installs every skill and connects Clonvo's MCP server (`web_search` and `web_scrape`) using `CLONVO_API_KEY` from your environment:

```
/plugin marketplace add clonvodotcom/skills
/plugin install clonvo@clonvo
```

### GitHub CLI

```bash
gh skill install clonvodotcom/skills
```

## Docs

Every Clonvo docs page is available as Markdown for agents: start at [clonvo.com/llms.txt](https://clonvo.com/llms.txt).
