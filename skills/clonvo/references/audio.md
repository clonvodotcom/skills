# Audio: speech to text and text to speech

`POST https://api.clonvo.com/v1/audio/transcriptions` and `POST https://api.clonvo.com/v1/audio/speech`, OpenAI's Audio API. The OpenAI SDKs work by changing the base URL; the key, credits, own provider keys and spending limits are the same as for chat.

Canonical docs: https://clonvo.com/docs/audio.md

## Models

Audio models aren't in the default `GET /v1/models` list. List them live:

```bash
curl -s "https://api.clonvo.com/v1/models?output_modalities=transcription" | jq -r '.data[].id'
curl -s "https://api.clonvo.com/v1/models?output_modalities=speech" | jq -r '.data[] | [.id, (.supported_voices | join(","))] | @tsv'
```

At the time of writing:

- Transcription: `openai/gpt-transcribe`, `google/gemini-3.5-transcribe`, `x-ai/grok-voice-transcribe-2.0`, `mistralai/voxtral-mini-transcribe-3-0`, `mistralai/voxtral-small-transcribe-3-0`, `qwen/qwen3-asr-flash`, `meta/muse-voice-transcribe-1.0`.
- Speech: `google/gemini-3.8-flash-tts`, `google/gemini-3.8-flash-lite-tts`, `x-ai/grok-voice-tts-1.0`, `mistralai/voxtral-mini-tts-3-0`, `qwen/qwen3-tts-flash`.

Pricing, as on OpenRouter: per-minute models list their price per second in `pricing.prompt`, per-character models their price per character. Gemini models are priced per token (`pricing.audio` for audio input tokens, `pricing.audio_output` for audio output tokens).

Audio models only serve these endpoints. One sent to chat completions returns `400 model_not_chat`.

## Speech to text

```python
with open("meeting.mp3", "rb") as f:
    transcription = client.audio.transcriptions.create(model="openai/gpt-transcribe", file=f)
print(transcription.text)
```

```ts
const transcription = await client.audio.transcriptions.create({
  model: "openai/gpt-transcribe",
  file: fs.createReadStream("meeting.mp3"),
});
```

Without a file, send JSON as on OpenRouter: `{"model": "x-ai/grok-voice-transcribe-2.0", "input_audio": {"data": "<base64>", "format": "mp3"}}`.

| Parameter | Notes |
| --- | --- |
| `file` / `input_audio` | Required. Multipart file, or base64 `data` plus `format`. Up to 25 MB, less on some models (`413 request_too_large`); Meta's model takes WAV only. |
| `model` | Required. A transcription model. |
| `language` | ISO-639-1 code (`en`, `fr`). Optional; most models detect it. |
| `prompt` | Words or context to expect. OpenAI only. |
| `response_format` | `json` (default), `verbose_json` (adds `language`, `duration`, and word timestamps on xAI) or `text`. Others, such as `srt`, return `400`. |
| `provider` | Extension: `order` and `allow_fallbacks` choose between own key and credits. |

```json
{
  "text": "Hello from Clonvo.",
  "usage": { "seconds": 5, "input_tokens": 0, "output_tokens": 0, "cost": 0.000375, "is_byok": false },
  "model": "openai/gpt-transcribe",
  "provider": "OpenAI"
}
```

## Text to speech

```python
speech = client.audio.speech.create(
    model="google/gemini-3.8-flash-tts",
    input="Your order has shipped.",
    voice="Kore",
    response_format="wav",
)
speech.write_to_file("order.wav")
```

| Parameter | Notes |
| --- | --- |
| `model` | Required. A speech model. |
| `input` | Required. Up to the model's limit (`400 input_too_long` over it). |
| `voice` | One of the model's `supported_voices`, case-insensitive. Defaults to its first voice. OpenAI's voice names (`alloy`, …) aren't valid. |
| `response_format` | One the model lists; defaults to its first. `pcm` is raw 16-bit little-endian samples. |
| `speed` | xAI only, 0.7 to 1.5. |
| `instructions` | Gemini only: how to say it ("cheerful and friendly"). |

The response body is the audio, with its `Content-Type`. Its cost isn't in the response: read it with `GET /v1/generation?id=` using the `x-clonvo-generation-id` header.

## Errors

| Status and code | Cause |
| --- | --- |
| `400 model_wrong_kind` | A chat model, or a speech model sent to transcriptions (or the reverse) |
| `400 unsupported_voice` | A voice the model doesn't have |
| `400 unsupported_parameter` | A `response_format` the model can't return |
| `400 unsupported_audio_format` | An input format the model can't read |
| `400 input_too_long` | Text over the model's limit |
| `400 missing_file`, `invalid_audio` | No audio, or base64 that doesn't decode |
| `413 request_too_large` | Audio over the size limit |

Not available: OpenAI's `audio.translations`, realtime and streamed speech.
