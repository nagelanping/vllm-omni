---
name: fish-speech-s2-pro
description: Deploy, serve, and use Fish Speech S2 Pro (fish-audio/s2-pro) via vLLM-Omni. Covers server startup, text-to-speech synthesis, inline [tag] timbre/prosody control, voice cloning with ref_audio, streaming, voice upload, and the Gradio web demo. Use this skill whenever the user mentions Fish Speech, S2 Pro, fishaudio, fish TTS, [voice tags], or wants to generate speech with vLLM-Omni.
---

# Fish Speech S2 Pro on vLLM-Omni

Fish Speech S2 Pro is a 4B Dual-AR text-to-speech model by FishAudio. It natively supports Chinese, English, and Japanese (Tier 1), with 80+ additional languages. Inline `[tag]` control over prosody, emotion, and delivery style uses **English** natural-language descriptions inside brackets. Chinese/Japanese tag descriptions are not supported — always write tags in English.

## Quick reference

The port depends on how the server is launched:

- **Direct `vllm serve`**: whatever `--port` you passed (vLLM-Omni default is 8091). Auth: `Bearer EMPTY`.
- **llama-swap**: dynamic port — find it with the command below, then substitute `<PORT>`. Auth: `Bearer Si.AI`.

```bash
# Find S2 Pro port (llama-swap)
journalctl -u llama-swap --no-pager -n 200 | grep -oP 'Health check passed on http://localhost:\K\d+'

# Direct launch (local model path)
CUDA_VISIBLE_DEVICES=0,1 vllm serve /home/Si/.local/share/ai/models/fishaudio/s2-pro --omni \
    --host 127.0.0.1 --port <PORT> --served-model-name fishaudio/s2-pro \
    --deploy-config vllm_omni/deploy/fish_qwen3_omni_2gpu_16gb.yaml

# API test
curl -s -X POST http://localhost:<PORT>/v1/audio/speech \
  -H "Content-Type: application/json" -H "Authorization: Bearer EMPTY" \
  -d '{"input": "Hello world.", "response_format": "wav"}' -o out.wav

# Gradio demo
python examples/online_serving/text_to_speech/fish_speech/gradio_demo.py --api-base http://localhost:<PORT>
```

## Server deployment

`fish_qwen3_omni_2gpu_16gb.yaml` is the only verified stable deploy config. Other YAML files in `vllm_omni/deploy/` are untested — use at your own risk.

The model is loaded from the **local checkout** at `/home/Si/.local/share/ai/models/fishaudio/s2-pro` (no HuggingFace download). `--served-model-name fishaudio/s2-pro` keeps the API-reported name stable regardless of the on-disk path.

```bash
CUDA_VISIBLE_DEVICES=0,1 vllm serve /home/Si/.local/share/ai/models/fishaudio/s2-pro --omni \
    --host 127.0.0.1 --port <PORT> --served-model-name fishaudio/s2-pro \
    --deploy-config vllm_omni/deploy/fish_qwen3_omni_2gpu_16gb.yaml
```

Stage 0 (Slow AR 4B) on GPU 0, Stage 1 (DAC decoder) on GPU 1. Data flows through shared memory. VRAM caps (0.90 / 0.25), concurrency (max 8 seqs), and sampling params are set in the YAML — no CLI override needed.

### Common serve flags

| Flag | Notes |
|------|-------|
| `--port` | API listen port |
| `--host` | Bind address (`127.0.0.1` behind llama-swap) |
| `--served-model-name` | Model name reported via API |
| `--deploy-config` | YAML path; required for multi-GPU |

## API usage

All requests go to `POST /v1/audio/speech`. Auth: `Bearer EMPTY` (or omit).

### Text-only synthesis (random timbre)

Fish Speech has **no built-in speakers**. When no reference audio is provided, the model picks a random voice. Timbre is steerable via inline `[tag]` syntax.

```bash
curl -s -X POST http://localhost:<PORT>/v1/audio/speech \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{"input": "Your text here.", "response_format": "wav"}' \
  -o output.wav
```

**Do NOT send `"voice": "default"`** — it will fail with "Unknown voice 'default'. Fish Speech has no built-in speakers."

### Inline `[tag]` control

S2 Pro accepts **free-form natural-language descriptions** inside `[brackets]` to control voice delivery at the word level. Tags are not a fixed enumeration — the model was trained on 15,000+ unique tags and generalizes to open-ended descriptions.

**Crucially, tags must be in English.** Even when the spoken text is Chinese, Japanese, or any other language, the `[tag]` description itself must be written in English.

This is essentially **inline voice design**: write anything from a single English keyword to a full English descriptive phrase inside `[brackets]`, and the model interprets the intent and renders accordingly.

**What tags can describe:** emotion, voice quality, character/persona, delivery style, paralinguistic sounds, volume/dynamics, phonetic guidance.

**Usage rules:**

- Tags precede the text segment they apply to
- Multiple tags can be stacked: `[tag1][tag2]text`
- A single `[tag]` can be a keyword or a full descriptive sentence
- Spoken-text languages can switch mid-input — but tags must stay in English
- Tags work identically in text-only and voice-cloning modes

### Voice cloning

Provide a reference audio clip (10-30s of clear speech) plus its exact transcript. The model clones the speaker's voice and synthesizes new content in that voice.

```bash
curl -s -X POST http://localhost:<PORT>/v1/audio/speech \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{
    "input": "New text to speak in the cloned voice.",
    "ref_audio": "data:audio/wav;base64,<base64_encoded_audio>",
    "ref_text": "Exact transcript of the reference audio.",
    "response_format": "wav"
  }' \
  -o cloned.wav
```

`ref_audio` accepts:
- `data:audio/wav;base64,...` (base64-encoded WAV, also supports mp3/flac/ogg)
- `https://...` (HTTP/HTTPS URL to audio file)
- `file:///path/to/audio.wav` (local file path, requires `--allowed-local-media-path`)

`ref_text` must be the **exact transcript** of what is spoken in the reference audio. Inline `[tag]` control works in `input` alongside voice cloning.

### Upload a voice (reusable speaker)

Register reference audio as a named voice, then reuse it by name without re-uploading:

```bash
# Upload
curl -X POST http://localhost:<PORT>/v1/audio/voices \
  -F "name=<voice_name>" \
  -F "consent=<recording_id>" \
  -F "audio_sample=@<reference_audio_file>" \
  -F "ref_text=<exact transcript of reference clip>"

# Use by name
curl -s -X POST http://localhost:<PORT>/v1/audio/speech \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{"input": "Your text.", "voice": "<voice_name>", "response_format": "wav"}' \
  -o output.wav

# List uploaded voices
curl -s http://localhost:<PORT>/v1/audio/voices
```

Audio file max 10MB. Supports wav/mp3/flac/ogg.

### Streaming

Set `"stream": true` and `"response_format": "pcm"` for progressive playback at 44.1kHz int16 PCM.

```bash
curl -s -X POST http://localhost:<PORT>/v1/audio/speech \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{"input": "Your text.", "stream": true, "response_format": "pcm"}' \
  -o stream.pcm
```

Works with both text-only and voice cloning modes.

### Request parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `input` | `str` | required | Text to synthesize; supports `[tag]` and multilingual |
| `voice` | `str` | — | Uploaded voice name. **Do not use** without prior upload |
| `ref_audio` | `str` | — | Base64 data URL, HTTP URL, or file:// path |
| `ref_text` | `str` | — | Required when `ref_audio` is present |
| `response_format` | `str` | `"wav"` | `wav`, `mp3`, `flac`, `pcm`, `aac`, `opus` |
| `stream` | `bool` | `false` | Enables PCM streaming (locks format to `pcm`) |
| `max_new_tokens` | `int` | — | Override max generated tokens (1–65536) |

### Response

- **200**: Audio binary in the requested format (WAV/MP3/FLAC/etc.) or PCM stream.
- **400**: Validation error (empty input, missing `ref_text` with `ref_audio`, unknown voice name, etc.).

## Gradio web demo

After starting the server, you can launch the UI for user to use. Activate the uv-managed venv first:

```bash
source /home/Si/.local/share/ai/vLLM-Omni/.venv/bin/activate
cd /home/Si/.local/share/ai/vLLM-Omni/vllm-omni
python examples/online_serving/text_to_speech/fish_speech/gradio_demo.py \
    --api-base http://localhost:<PORT> \
    --port 7860 \
    --host 0.0.0.0
```

Flags: `--api-base`, `--host` (default `0.0.0.0`), `--port` (default `7860`).

The demo supports text input, voice cloning (upload/microphone/URL), a streaming toggle, and format selection. Streaming uses a **same-origin FastAPI proxy** (`/proxy/v1/audio/speech`) feeding a **Web Audio API AudioWorklet** player for gap-free playback — this replaces gradio's built-in streaming `gr.Audio`, which has inherent inter-chunk gaps/clicks. When a stream finishes, the browser assembles the accumulated PCM into a WAV so the full clip stays playable and downloadable. Non-streaming still returns the full clip via `gr.Audio`.

## Key gotchas

1. **No `voice: "default"`**: Fish Speech has zero built-in speakers. Either omit `voice` entirely (random timbre with `[tag]` control), or use `ref_audio`/uploaded voice.
2. **`ref_audio` must have `ref_text`**: Voice cloning always requires the transcript. Without it, the request fails with a 400 error.
3. **PCM is 44.1kHz int16**: When decoding streaming PCM output, use `np.frombuffer(data, dtype=np.int16)` / 32767.0 to float32.
4. **Streaming forces PCM**: `"stream": true` locks `response_format` to `pcm`. WAV/MP3 cannot be streamed (they need a header first).
5. **Two stages internally**: Slow AR (4B, semantic tokens) to DAC decoder (residual codebooks to audio). Both are served by the same `vllm serve` process; the deploy YAML wires them together via shared memory.
