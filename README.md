# Clonvo agent skills

[![skills.sh](https://skills.sh/b/clonvodotcom/skills)](https://skills.sh/clonvodotcom/skills)

[Agent Skills](https://agentskills.io) for [Clonvo](https://clonvo.com), the AI gateway for agents: 150+ models from OpenAI, Anthropic, Google Gemini, xAI Grok, DeepSeek, Mistral, Kimi, Meta and Qwen, plus web search and scraping, crawls, research, a browser agent, and company and people data, on one API key. With fallbacks, your own provider keys, and the OpenAI SDKs you already use.

They work with any agent that supports skills, including Claude Code, Cursor, Codex, GitHub Copilot, Gemini CLI, Windsurf and OpenCode.

> [!NOTE]
> Skills that call the API need a Clonvo API key. Create one in the [Clonvo dashboard](https://clonvo.com) and set it as `CLONVO_API_KEY` in your agent's environment. The MCP server lets you sign in instead.

## Skills

| Skill | Use it to |
| --- | --- |
| [`clonvo`](skills/clonvo/SKILL.md) | Build with the API: chat completions, the Responses API, embeddings, speech to text and text to speech, decision models, streaming, tool calling, structured outputs, reasoning, images and PDFs, fallback models, your own provider keys, web tools, errors and billing. |
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

The plugin installs every skill and connects [Clonvo's MCP server](https://github.com/clonvodotcom/mcp) (web search, scraping, crawling, research, a browser agent, and company and people data):

```
/plugin marketplace add clonvodotcom/skills
/plugin install clonvo@clonvo
```

Then run `/mcp` and choose **Authenticate** for `clonvo` to sign in. To use an API key instead, see [the MCP server's setup](https://github.com/clonvodotcom/mcp#claude-code).

### GitHub CLI

```bash
gh skill install clonvodotcom/skills
```

## Docs

Every Clonvo docs page is available as Markdown for agents: start at [clonvo.com/llms.txt](https://clonvo.com/llms.txt).

## License

[MIT](LICENSE)
