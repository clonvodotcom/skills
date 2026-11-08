---
name: clonvo-models
description: "Find, compare and price the models available on Clonvo from its live catalogue: list models, resolve an informal name to an exact model id, compare prices and context lengths, find the cheapest model with tools, vision, reasoning or structured outputs, list free and embedding models, and check peak-hour pricing. Use when choosing a Clonvo model id, when the user asks which models Clonvo has or what they cost, or before writing a model id into code."
metadata:
  author: Clonvo
  version: "1.0.0"
  docs: "https://clonvo.com/docs/models.md"
---

# Clonvo models

The catalogue is live at `GET https://api.clonvo.com/v1/models`: public, no key, OpenRouter's format, cached up to a minute. Always read it instead of relying on memory; models are added and retired.

The recipes below use `curl` and `jq`. Save the list once per task and query the file:

```bash
curl -s https://api.clonvo.com/v1/models -o /tmp/clonvo-models.json
```

No `jq`? `python3 -c 'import json; …'` over the same file works.

## Decision table

| User wants to… | Recipe |
| --- | --- |
| See every model | List models |
| Turn "sonnet", "the new gemini flash" into an id | Resolve a name |
| Compare a few models | Compare models |
| Cheapest model that can do X | Filter by capability |
| Free models | Free models |
| Embedding models | Embedding models |
| Details or price windows for one model | One model |

## List models

```bash
jq -r '.data[] | [.id, .name, .context_length, (.pricing.prompt|tonumber*1e6*1e4|round/1e4), (.pricing.completion|tonumber*1e6*1e4|round/1e4)] | @tsv' /tmp/clonvo-models.json \
  | sort | column -t -s $'\t'
```

Columns: id, name, context length, input and output USD per 1M tokens. Prices in the API are per token, as strings.

Newest first: sort by `.created` (release date, Unix seconds):

```bash
jq -r '.data | sort_by(-.created)[] | [(.created|todate[:10]), .id] | @tsv' /tmp/clonvo-models.json | head -20
```

## Resolve a name

Search ids and names, case-insensitively:

```bash
jq -r --arg q "sonnet" '.data[] | select((.id + " " + .name) | ascii_downcase | contains($q)) | [.id, .name, (.created|todate[:10])] | @tsv' /tmp/clonvo-models.json
```

- One match: use it.
- Several (e.g. versions): prefer the newest unless the user named a version, and say which you picked.
- None: search a shorter term; never invent an id. Clonvo also accepts aliases (`xai/…`, `gemini/…`), bare slugs and `.`/`-` variants, but write the canonical `id` into code.

## Compare models

```bash
jq -r --argjson ids '["anthropic/claude-sonnet-5-5","openai/gpt-6.1-sol","google/gemini-3.8-flash"]' '
  .data[] | select(.id as $id | $ids | index($id))
  | [.id, .context_length, .top_provider.max_completion_tokens,
     (.pricing.prompt|tonumber*1e6*1e4|round/1e4), (.pricing.completion|tonumber*1e6*1e4|round/1e4),
     (.architecture.input_modalities|join(",")),
     ([.supported_parameters[] | select(. == "tools" or . == "structured_outputs" or . == "reasoning")] | join(","))] | @tsv' /tmp/clonvo-models.json
```

Columns: id, context, max output, input $/1M, output $/1M, input modalities, key capabilities.

## Filter by capability

| Capability | Test |
| --- | --- |
| Tool calling | `.supported_parameters \| index("tools")` |
| JSON Schema output | `.supported_parameters \| index("structured_outputs")` |
| Reasoning control | `.supported_parameters \| index("reasoning")` |
| Image input | `.architecture.input_modalities \| index("image")` |
| PDF input | `.architecture.input_modalities \| index("file")` |
| Audio input | `.architecture.input_modalities \| index("audio")` |
| Context ≥ N | `.context_length >= N` |

Cheapest paid models with tools and image input, by output price:

```bash
jq -r '.data[]
  | select(.supported_parameters | index("tools"))
  | select(.architecture.input_modalities | index("image"))
  | select((.pricing.completion|tonumber) > 0)
  | [(.pricing.completion|tonumber*1e6*1e4|round/1e4), (.pricing.prompt|tonumber*1e6*1e4|round/1e4), .context_length, .id] | @tsv' /tmp/clonvo-models.json | sort -n | head
```

Estimate a request's cost: `prompt_tokens × pricing.prompt + completion_tokens × pricing.completion` (plus cache, `request` and `web_search` charges where non-zero). The real cost of every request is in its `usage.cost`.

## Free models

```bash
jq -r '.data[] | select(.id | endswith(":free")) | [.id, .context_length] | @tsv' /tmp/clonvo-models.json
```

Before recommending one, tell the user: an owner or admin must turn on **Use free models** in **Settings → Privacy**; free-model prompts are stored by Clonvo for up to 12 months and may be used for training; limits are 20 requests a minute and 50 a day per organization (1,000 a day after $10 of purchases). Don't use them for confidential data.

## Embedding models

Not in the default list:

```bash
curl -s https://api.clonvo.com/v1/embeddings/models | jq -r '.data[] | [.id, .context_length, (.pricing.prompt|tonumber*1e6*1e4|round/1e4), (.supported_parameters|join(","))] | @tsv'
```

`dimensions` in `supported_parameters` means the model can shorten its vectors.

## One model

```bash
curl -s https://api.clonvo.com/v1/models/anthropic/claude-sonnet-5-5 | jq '.data'
```

`404 model_not_found` means it doesn't exist or isn't available now. The human-readable page, with deprecation dates, per-model parameters and reasoning efforts, is `https://clonvo.com/models/<author>/<slug>.md`.

Peak-hour prices: models with `.pricing.overrides` cost more in some UTC windows (`utc_days`, `utc_start`/`utc_end` as HHMM). The top-level price is off-peak.

```bash
jq -r '.data[] | select(.pricing.overrides) | .id' /tmp/clonvo-models.json
```

## Notes

- Retired and switched-off models aren't listed. A model page may still show a retired model with its retirement date.
- Some providers run only on the user's own key (`400 provider_requires_own_key` without one); the dashboard marks those models **Needs your own key**.
- For using the chosen model in code, see the `clonvo` skill.
