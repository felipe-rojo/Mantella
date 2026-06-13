---
name: mantella-config
description: "Domain knowledge for Mantella configuration system. Covers: ConfigLoader, ConfigValue types, config groups, config definitions, config constraints, config file writing, config JSON export, model profiles, per-character overrides, visitor pattern, config.ini lifecycle. Load when working on config features."
---

# Mantella Config Domain Knowledge

## Architecture

```
config.ini → ConfigLoader → ConfigValue objects → consumed by all subsystems
                ↓
         ConfigValueVisitor → SettingsUIConstructor (Gradio UI)
                ↓
         ConfigFileWriter → config.ini (persisted)
                ↓
         ConfigJsonWriter → JSON (API / UI)
```

**ConfigLoader** (`src/config/config_loader.py`) is the central hub. On init it:
1. Instantiates all `ConfigValue` definitions via `MantellaConfigValueDefinitionsNew`
2. If `config.ini` doesn't exist, writes defaults via `ConfigFileWriter`
3. Reads `config.ini` with `configparser`, parsing each key into its `ConfigValue`
4. On parse failure, creates a backup (`config_backup_N.ini`) and logs a warning
5. Calls `__update_config_values_from_current_state()` to populate convenience properties

## Key Files

| File | Purpose |
|------|---------|
| `src/config/config_loader.py` | `ConfigLoader` — loads config.ini, exposes all settings as properties |
| `src/config/config_values.py` | `ConfigValues` — visitor-based registry of all config values |
| `src/config/config_value_constraint.py` | `ConfigValueConstraint` base + `ConfigValueConstraintResult` |
| `src/config/config_file_writer.py` | `ConfigFileWriter` — writes config.ini from ConfigValue objects |
| `src/config/config_json_writer.py` | `ConfigJsonWriter` — writes config as JSON for API/UI |
| `src/config/config_editor.py` | Top-level config editor utilities |
| `src/config/definitions/*.py` | Per-group config value definitions (11 files) |
| `src/config/types/*.py` | Config value type implementations (10 types) |
| `src/model_profile_manager.py` | `ModelProfileManager` — per-model parameter profiles |
| `src/config_editor.py` | Top-level config editor |

## Config Value Types

All types extend `ConfigValue[T]` (`src/config/types/config_value.py`):

| Type | Class | INI Widget | Notes |
|------|-------|------------|-------|
| `int` | `ConfigValueInt` | number input | Has `min_value`, `max_value` |
| `float` | `ConfigValueFloat` | number input | Has `min_value`, `max_value` |
| `bool` | `ConfigValueBool` | checkbox | Parses "True"/"False" |
| `str` | `ConfigValueString` | text input | Multiline values JSON-encoded in INI |
| `selection` | `ConfigValueSelection` | dropdown | Optional enum mapping; `allows_free_edit` / `allows_values_not_in_options` |
| `path` | `ConfigValuePath` | text + file picker | Validates file/folder presence |
| `multi-selection` | `ConfigValueMultiSelection` | checkbox group | Stored as comma-separated values |
| `group` | `ConfigValueGroup` | tab/container | Holds child `ConfigValue` objects |

**Tags** (`ConfigValueTag`): `advanced` — shown in advanced section; `share_row` — shares UI row.

## Config Groups (11 tabs)

Each group has a definitions file in `src/config/definitions/`:

| Group | File | Key Settings |
|-------|------|-------------|
| **Game** | `game_definitions.py` | `game`, `skyrim_mod_folder`, `fallout4_mod_folder`, etc. |
| **LLM** | `llm_definitions.py` | `llm_api`, `model`, `max_response_sentences_single/multi`, `llm_params`, `custom_token_count`, `narration_handling`, `narration_indicators`, `claude_prompt_caching_enabled`, `random_llm_*`, `summary_llm_*` |
| **TTS** | `tts_definitions.py` | `tts_service`, `xvasynth_folder`, `xtts_server_folder`, `piper_folder`, `lipgen_folder`, `facefx_folder`, `number_words_tts`, `lip_generation` |
| **STT** | `stt_definitions.py` | `stt_service`, `audio_threshold`, `pause_threshold`, `ptt_enabled`, `moonshine_model_size`, `whisper_model_size`, `stt_language`, `silence_auto_response_*` |
| **Vision** | `vision_definitions.py` | `vision_enabled`, `low_resolution_mode`, `image_quality`, `custom_vision_model`, `vision_llm_api`, `vision_model`, `vision_llm_params` |
| **Actions** | `action_definitions.py` | `advanced_actions_enabled`, `disabled_actions`, `custom_function_model`, `function_llm_*` |
| **Language** | `language_definitions.py` | `language`, `end_conversation_keyword`, `goodbye_npc_response`, `collecting_thoughts_npc_response` |
| **Prompts** | `prompt_definitions.py` | `skyrim_prompt`, `fallout4_prompt`, `multi_npc_prompt`, `radiant_prompt`, `memory_prompt`, `vision_prompt`, `function_llm_prompt` |
| **Startup** | `startup_definitions.py` | `auto_launch_ui`, `play_startup_sound`, `remove_mei_folders` |
| **Profiles** | `model_profile_definitions.py` | `apply_model_profiles`, `profile_selected_service`, `profile_selected_model`, `profile_parameters` |
| **Other** | `other_definitions.py` | `automatic_greeting`, `max_count_events`, `events_refresh_time`, `hourly_time`, `player_character_description`, `voice_player_input`, `conversation_summary_enabled`, `enable_character_tag_reading` |

## Visitor Pattern

`ConfigValues` implements `ConfigValueVisitor`. Each `ConfigValue` subclass implements `accept_visitor()` which calls the corresponding `visit_*` method. Used by:
- `ConfigValues` — registers values into typed dictionaries on visit
- `ConfigFileWriter` — serializes to INI format
- `ConfigJsonWriter` — serializes to JSON
- `SettingsUIConstructor` — builds Gradio UI

## ConfigValue Base Class

```python
class ConfigValue(ABC, Generic[T]):
    identifier: str      # INI key (e.g. "game")
    name: str            # Display name (e.g. "Game")
    description: str     # Help text
    value: T             # Current value
    default_value: T     # Default value
    constraints: list[ConfigValueConstraint[T]]
    is_hidden: bool      # Hidden from UI
    tags: list[ConfigValueTag]
    
    parse(config_value: str) -> ConfigValueConstraintResult  # Abstract
    accept_visitor(visitor: ConfigValueVisitor)              # Abstract
    does_value_cause_error(value) -> ConfigValueConstraintResult
```

## Constraints

`ConfigValueConstraint[T]` is an abstract base with `apply_constraint(value) -> ConfigValueConstraintResult`. Built-in constraint examples:
- `GameDefinitions.ProgramFilesChecker` — warns if game installed in Program Files
- `GameDefinitions.ModFolderChecker` — validates Mantella.esp exists in mod folder
- `TTSDefinitions.ResourceFolderExistsChecker` — validates xVASynth resources subfolder
- `STTDefinitions.WhisperProcessDeviceChecker` — validates whisper device

Constraints are evaluated on `parse()` and on `get_*_value()` calls. Failures are collected in `ConfigValues.constraint_violations`.

## INI File Format

Sections map to `ConfigValueGroup` identifiers. Format:
```ini
[game]
; game
;   Choose the game to run with Mantella.
;   Options: Skyrim, SkyrimVR, Fallout4, Fallout4VR
;   default = SkyrimVR
game = SkyrimVR
```

Special handling:
- Hash symbols (`#`) are escaped as `\#` on write, unescaped on read
- Multiline string values are JSON-encoded to preserve whitespace
- Backups created as `config_backup_N.ini` on parse errors

## Model Profiles

`ModelProfileManager` (`src/model_profile_manager.py`) stores per-model LLM parameter overrides in `data/model_profiles.json`.

- **Key**: `{endpoint_url}:{model}` (e.g. `https://openrouter.ai/api/v1:google/gemma-2-9b-it:free`)
- **Resolution**: Service names resolved via `utils.resolve_service_endpoint()` — aliases like "openrouter", "or", "OR" all map to the same endpoint
- **CRUD**: `create_or_update_profile()`, `get_profile()`, `has_profile()`, `delete_profile()`
- **Application**: `resolve_params(service, model, fallback_params, apply_profile)` — returns profile params if profile exists and `apply_profile=True`, else returns fallback
- **Singleton**: `get_profile_manager()` returns a module-level singleton
- **Rollback**: Failed saves roll back in-memory changes

## Per-Character Overrides

Config values controlling per-character override behavior:
- `allow_per_character_tts_overrides` — enables per-character TTS service selection via CSV `tts_service` column
- `allow_per_character_llm_overrides` — enables per-character LLM model selection via CSV columns

## Config Change Detection

- `ConfigLoader.has_any_config_value_changed` — set to `True` when any `ConfigValue` changes (via callback)
- `update_config_loader_with_changed_config_values()` — re-reads all values from definitions, resets flag
- Auto-writes `config.ini` on value change (unless during initial load)

## Adding a New Config Value

1. Add definition method in appropriate `src/config/definitions/*_definitions.py`
2. Register in `MantellaConfigValueDefinitionsNew.get_config_values()` under the correct group
3. Add property in `ConfigLoader.__update_config_values_from_current_state()`
4. Add constraints if validation needed
5. Run tests: `tests/config/test_config_loader.py`, `tests/config/test_config_file_writer.py`

## Common Patterns

- **Enum-backed selections**: `ConfigValueSelection` accepts `corresponding_enums` list; `ConfigLoader` uses `get_enum_value()` to return the enum
- **JSON config values**: `llm_params`, `vision_llm_params`, `function_llm_params` stored as JSON strings, parsed with `json.loads()` in ConfigLoader
- **Conditional validation**: Path values only validated when the corresponding service is selected (e.g. `validate_xtts_path` only when `tts_service == XTTS`)
- **Game-specific prompts**: Fallout 4 uses `fallout4_prompt` / `fallout4_multi_npc_prompt`; Skyrim uses `skyrim_prompt` / `skyrim_multi_npc_prompt`
- **Integrated mode**: When `--integrated` in `sys.argv`, paths are resolved relative to the exe location instead of from config

## Tests

- `tests/config/test_config_loader.py` — init, change detection, definitions property
- `tests/config/test_config_file_writer.py` — hash escaping, multiline preservation, round-trip
- `tests/test_model_profile_manager.py` — CRUD, endpoint keying, resolve_params, singleton
