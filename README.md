# Clonvo agent skills

Agent skills for [Clonvo](https://clonvo.com). Each skill is a folder under `skills/` with a `SKILL.md` that tells a coding agent (Claude Code, Cursor, Codex, Copilot, Gemini CLI and others) how to use Clonvo.

| Skill | What it does |
| --- | --- |
| [clonvo](skills/clonvo/SKILL.md) | Call OpenAI, Anthropic, Gemini, Grok, DeepSeek, Mistral, Kimi and Meta models through Clonvo's one OpenAI-compatible API, with fallback models and web search and scraping tools for agents. |

## Install

```bash
npx skills add clonvodotcom/skills --skill clonvo
```

Browse it on [skills.sh](https://skills.sh/clonvodotcom/skills/clonvo).
