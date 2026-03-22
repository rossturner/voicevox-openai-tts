# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A FastAPI server that bridges VOICEVOX and AivisSpeech TTS engines with the OpenAI TTS API format. Clients using the OpenAI SDK can point to this server to use Japanese TTS voices.

## Running the Server

```bash
# CPU (VOICEVOX)
docker compose up -d

# GPU (VOICEVOX)
docker compose -f docker-compose.gpu.yml up -d

# AivisSpeech with bundled engine
docker compose -f docker-compose.aivis-speech.yml up -d

# AivisSpeech API bridge (requires local AivisSpeech running on port 10101)
docker compose -f docker-compose.aivis-speech-api-only.yml up -d
```

The API server listens on port 8000. VOICEVOX engine runs on port 50021; AivisSpeech on port 10101.

## Testing

No automated test suite — manual testing via examples:

```bash
pip install -r example/requirements.txt
python example/simple_tts_example.py   # basic smoke test
python example/tts_example.py          # comprehensive test suite
```

## Architecture

**Request flow:**
```
POST /v1/audio/speech (OpenAI format)
  → voice name lookup in voice_mappings/*.json
  → VOICEVOX/AivisSpeech /audio_query (text → acoustic features)
  → /synthesis (acoustic features → WAV → MP3)
  → Response: audio/mpeg
```

**Key files:**
- `voicevox_tts_api/api/routers/speech.py` — core TTS logic; handles voice mapping, engine communication, audio synthesis
- `voicevox_tts_api/api/routers/chat.py` — stub chat completions endpoint (returns dummy responses for OpenAI client compatibility)
- `voice_mappings/voicevox.json` and `voice_mappings/aivis-speech.json` — map OpenAI voice names (alloy, echo, etc.) to engine speaker IDs

**Environment variables:**
- `VOICEVOX_ENGINE_URL` — base URL for the TTS engine (default: `http://voicevox_engine:50021`)
- `VOICE_MAPPINGS_PATH` — path to voice mappings JSON (default: `/app/voice_mappings.json`)

## Preferred Voices

Pass the speaker ID as the `voice` parameter (numeric strings are accepted directly):

| Use | Character | Style | Speaker ID |
|---|---|---|---|
| Female | 波音リツ (Namine Ritsu) | ノーマル | `"9"` |
| Male | 雀松朱司 (Suzume Shuji) | ノーマル | `"52"` |

```python
client.audio.speech.create(model="voicevox-v1", voice="9", input="...")   # female
client.audio.speech.create(model="voicevox-v1", voice="52", input="...")  # male
```

## Dependencies

Pinned to narrow version ranges in `voicevox_tts_api/requirements.txt`:
- FastAPI `<0.69.0`, Pydantic `<2.0.0`, uvicorn `<0.16.0`

Pydantic v1 API is in use throughout — avoid v2 syntax when modifying schemas.

## CI/CD

GitHub Actions (`.github/workflows/docker-image.yml`) builds and publishes multi-arch Docker images (`linux/amd64`, `linux/arm64`) to GHCR on pushes to main and version tags.
