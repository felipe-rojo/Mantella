---
name: mantella-tts
description: "Domain knowledge for Mantella Text-to-Speech subsystem. Covers: TTSable base class, Piper/XTTS/xVASynth providers, voiceline generation, lip sync, voice model resolution, TTS configuration, per-character overrides. Load when working on TTS features."
---

# Mantella TTS Domain Knowledge

## TTS Provider Architecture

All TTS providers extend `TTSable` (`src/tts/ttsable.py`):
- `synthesize()` — main orchestration (voice change → synthesis → lip generation → file rename)
- `change_voice()` — abstract, per-provider voice model loading
- `tts_synthesize()` — abstract, per-provider audio generation

## Providers

### Piper (`src/tts/piper.py`)
- Local subprocess: `piper.exe` via stdin/stdout
- Voice models: `.onnx` files in `<piper_path>/models/<game>/low/`
- Text preprocessing: `!` → `.` (non-aggro), `.` → `!` (aggro), strips `*` and newlines
- Retry/crash recovery: up to 5 attempts

### XTTS (`src/tts/xtts.py`)
- HTTP REST to XTTS-V2 API server (default `http://127.0.0.1:8020`)
- Auto-launches `xtts-api-server-mantella.exe` if local and not running
- Supports local and remote (RunPod) servers
- Models: `main`, `v2.0.3`, `v2.0.2`, `v2.0.1`, `v2.0.0`

### xVASynth (`src/tts/xvasynth.py`)
- HTTP REST to `http://127.0.0.1:8008`
- Phrase splitting (by commas, "and", "or") for long voicelines
- Batch synthesis for multi-phrase lines
- Auto-restarts server (up to 15 attempts)
- Model path: `<xvasynth_path>/resources/app/models/<GameName>/` with `sk_` or `f4_` prefix

## Voice Model Resolution

Priority: `advanced_voice_model` > `tts_voice` > `in_game_voice_model` > `csv_in_game_voice_model` > game's `find_best_voice_model()` (race-based dictionary)

## Voiceline Storage

- Temp: `<TMP>/voicelines/`
- Saved: `<TMP>/voicelines/save/` (uniquely named, deduplicated)
- Naming: `{voice} {voiceline}`[:150]

## Lip Sync

- Primary: LipGen (Bethesda tool)
- Fallback: FaceFXWrapper
- Config: `lip_generation` = Enabled/Lazy/Disabled
- Fallout 4: also generates `.fuz` files

## Per-Character Overrides

- Config: `allow_per_character_tts_overrides`
- CSV column: `tts_service`
- `ChatManager._get_or_create_tts()` caches per-character instances

## Key Config Values

`tts_service`, `piper_folder`, `xvasynth_folder`, `xtts_server_folder`, `xtts_url`, `xtts_default_model`, `xtts_device`, `xtts_deepspeed`, `xtts_lowvram`, `xtts_data`, `xtts_accent`, `tts_process_device`, `pace`, `use_sr`, `use_cleanup`, `lip_generation`, `number_words_tts`, `tts_print`, `allow_per_character_tts_overrides`, `fast_response_mode`, `fast_response_mode_volume`, `lipgen_folder`, `facefx_folder`

## Tests

- `tests/tts/test_piper.py` — voice change, synthesis retry, newline stripping
- `tests/tts/test_tts_factory.py` — service parsing
