---
name: mantella-stt
description: "Domain knowledge for Mantella Speech-to-Text subsystem. Covers: Transcriber class, Moonshine/Whisper providers, audio capture, Silero VAD, push-to-talk, transcription pipeline, STT configuration. Load when working on STT features."
---

# Mantella STT Domain Knowledge

## STT Architecture

Single monolithic `Transcriber` class (`src/stt/stt.py`) with internal branching:
- No abstract base class
- Branches between Moonshine and Whisper in `__init__()`
- Background thread for audio processing
- Thread-safe result delivery via `threading.Event`

## Providers

### Moonshine (default)
- Library: `moonshine_onnx` (from GitHub: `usefulsensors/moonshine`)
- English only
- Models: `moonshine/tiny/quantized`, `moonshine/tiny/quantized_4bit`, `moonshine/tiny/float`, `moonshine/base/quantized`, `moonshine/base/quantized_4bit`, `moonshine/base/float`
- Loads from local `moonshine_folder` or downloads from Hugging Face
- Graceful fallback if `moonshine_onnx` not installed

### Whisper (faster_whisper)
- Library: `faster_whisper==1.0.3`
- Multi-language support
- 17+ model variants (tiny through large-v3, distil variants, Numbat Skyrim-specific)
- External API mode: OpenAI, Groq, whisper.cpp
- API key from `secret_keys.json` or `GPT_SECRET_KEY.txt`
- Supports `task="translate"` for translation to English

## Audio Capture

- `sounddevice.InputStream`: 16kHz, mono, 512-sample blocks (32ms), float32, low latency
- Callback → `queue.Queue` → background thread

## Processing Modes

### VAD Mode (default)
- Silero VAD per chunk
- Speech start: `probability > audio_threshold`
- Speech end: below threshold for `pause_threshold` seconds
- Lookback: 5 chunks (2560 samples) before speech onset

### Proactive Mode
- Re-transcribes every `min_refresh_secs` while speech continues
- Config: `proactive_mic_mode`

### PTT Mode
- `PTTController` (`src/stt/ptt_controller.py`) — `GetAsyncKeyState` polling
- Accumulates audio while hotkey held, transcribes on release
- Config: `ptt_enabled`, `ptt_hotkey`

## Transcription Result

- `get_latest_transcription()` blocks on `threading.Event`
- Supports `silence_timeout` — returns `None` if no speech
- Clears state after each successful transcription

## Key Config Values

`stt_service`, `moonshine_model_size`, `moonshine_folder`, `whisper_model_size`, `process_device`, `stt_language`, `stt_translate`, `audio_threshold`, `proactive_mic_mode`, `min_refresh_secs`, `pause_threshold`, `listen_timeout`, `ptt_enabled`, `ptt_hotkey`, `play_cough_sound`, `save_mic_input`, `external_whisper_service`, `whisper_url`, `allow_interruption`, `silence_auto_response_enabled`, `silence_auto_response_timeout`, `silence_auto_response_max_count`, `silence_auto_response_message`

## Tests

- `tests/stt/test_ptt_controller.py` — 16 test cases for key normalization and controller lifecycle
- No Transcriber tests exist
