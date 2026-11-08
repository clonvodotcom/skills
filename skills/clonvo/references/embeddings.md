# Embeddings

`POST https://api.clonvo.com/v1/embeddings`, OpenAI's request and response shape.

Canonical docs: https://clonvo.com/docs/embeddings.md

## Models

List them live; they aren't in the default `GET /v1/models` list:

```bash
curl -s https://api.clonvo.com/v1/embeddings/models | jq -r '.data[] | [.id, .context_length, (.pricing.prompt | tonumber*1e6*1e4|round/1e4)] | @tsv'
```

At the time of writing: `openai/text-embedding-3-small`, `openai/text-embedding-3-large`, `google/gemini-embedding-2`, `mistralai/mistral-embed`, `mistralai/codestral-embed`. `supported_parameters` includes `dimensions` on models that can shorten their vectors.

## Request

```python
result = client.embeddings.create(
    model="openai/text-embedding-3-small",
    input=["How do I reset my password?", "Where can I see my invoices?"],
)
vectors = [item.embedding for item in result.data]  # in input order
```

| Parameter | Notes |
| --- | --- |
| `model` | Required. An embedding model; a chat model returns `400 model_not_embedding`. |
| `input` | Required. A string or up to 2,048 non-empty strings. Token-id arrays return `400`. |
| `encoding_format` | `float` (default) or `base64` (little-endian float32, much smaller). |
| `dimensions` | Within the model's range; fixed-size models accept only their size (`400 invalid_dimensions`). |
| `provider` | Extension: `order`/`allow_fallbacks` between own key and credits, as for chat. |

- No fallback `models` here: one model per request.
- Requests over 4 MB return `413`. Each input must fit the model's input limit (`context_length`); chunk long documents first.
- The response has `usage.prompt_tokens` and `usage.cost`, plus the generation id.

## Rules

- Embed documents and queries with the same model and the same `dimensions`; vectors from different models aren't comparable.
- Sending an embedding model to chat completions returns `400 embedding_model_not_chat`.
- LangChain's `OpenAIEmbeddings`: set `check_embedding_ctx_length=False`, otherwise it sends token ids, which Clonvo rejects.
