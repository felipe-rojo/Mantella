---
name: mantella-game
description: "Domain knowledge for Mantella game-specific logic. Covers: Gameable ABC, Skyrim/Fallout4 implementations, game detection, voice model resolution, character loading, equipment parsing, external character info, bio templates, character/characters manager, game state management. Load when working on game features."
---

# Mantella Game Domain Knowledge

## Architecture

```
Gameable (ABC)
├── Skyrim
└── Fallout4

GameStateManager
├── uses Gameable (polymorphic)
├── loads Character objects via game.load_external_character_info()
├── manages Conversation lifecycle
└── dispatches sentences to game.prepare_sentence_for_game()

Characters (in characters_manager.py)
├── tracks active NPCs in conversation
├── tracks participation log (join/leave events)
├── manages nearby NPCs and pending conversation shares
└── provides last_added_character for action targeting
```

## Key Files

| File | Purpose |
|------|---------|
| `src/games/gameable.py` | `Gameable` abstract base class — character CSV loading, override system, voice folder creation, NPC matching |
| `src/games/skyrim.py` | `Skyrim` — SKSE extender, weather table, idles, bio templates, XVASynth/Piper/XTTS voice dictionaries, VOICE_MODEL_IDS |
| `src/games/fallout4.py` | `Fallout4` — F4SE extender, FO4_Voice_folder_XVASynth_matches.csv, 448-byte text limit, robot voice workarounds |
| `src/games/equipment.py` | `Equipment` / `EquipmentItem` — slot-based equipment with natural language description |
| `src/games/external_character_info.py` | `external_character_info` — DTO for character bios, voice models, per-character LLM/TTS overrides |
| `src/game_manager.py` | `GameStateManager` — conversation lifecycle, character loading, context updates, sentence dispatch |
| `src/character_manager.py` | `Character` — full NPC/player data model with voice, equipment, custom values, pronoun helpers |
| `src/characters_manager.py` | `Characters` — conversation NPC list with participation tracking, nearby NPCs, share queue |
| `src/bio_template_manager.py` | `BioTemplateManager` — tag-based bio template injection with 3-tier override cascade |

## Game Detection

Game detection happens in `ConfigLoader.__update_config_values_from_current_state()` (`src/config/config_loader.py`):

1. Resolves `game_path` from exe location: `Plugins/MantellaSoftware` → parent → parent → parent
2. Checks parent folder name for `vr`, `fallout`, `skyrim` keywords
3. Sets `config.game` to one of: `GameEnum.SKYRIM`, `SKYRIM_VR`, `FALLOUT4`, `FALLOUT4_VR`
4. VR variants set `is_vr = True` on the game instance

`GameEnum.base_game` maps VR variants to their base game (e.g., `SKYRIM_VR` → `SKYRIM`).

## Gameable Base Class

`Gameable` (`src/games/gameable.py`) provides shared logic:

- **Character CSV loading**: Reads `<game>_characters.csv` with encoding detection
- **Character overrides**: Two-tier system — mod overrides (`<mod_path>/<extender>/Plugins/MantellaSoftware/data/<game>/character_overrides/`) and personal overrides (`<save_folder>/data/<game>/character_overrides/`). Supports both `.json` and `.csv` override files.
- **NPC matching** (`_get_matching_df_rows_matcher`): Priority-ordered matching — name+ID+race → name+ID → name+partialID+race → name+partialID → name+race → name → ID
- **NPC ref_id resolution** (`resolve_npc_refid_by_name`): Only resolves if exactly one match exists (avoids ambiguity for guards, etc.)
- **Voice folder creation** (`_create_all_voice_folders`): Copies `MantellaVoice00` template to all voice folders in the mod directory

### Abstract Methods (per-game)

| Method | Purpose |
|--------|---------|
| `game_name_in_filepath` | `'skyrim'` or `'fallout4'` |
| `extender_name` | `'SKSE'` or `'F4SE'` |
| `image_path` | Screenshot path for vision |
| `modify_sentence_text_for_game` | Text truncation per game limits |
| `load_external_character_info` | Load bio + voice from CSV/external sources |
| `prepare_sentence_for_game` | Write audio/lip files to game folders |
| `is_sentence_allowed` | Content filtering |
| `load_unnamed_npc` | Generic NPC fallback |
| `get_weather_description` | Weather → prose for LLM prompts |
| `find_best_voice_model` | Race/gender → TTS voice model |

## Skyrim Implementation

**Extender**: SKSE  
**Character CSV**: `data/Skyrim/skyrim_characters.csv`  
**Voice folder column**: `skyrim_voice_folder`

### Voice Model Resolution (`find_best_voice_model`)

1. Parse `actor_voice_model_id` from parentheses, `actor_voice_model_name` from angle brackets
2. Select dictionary based on TTS service (XVASynth / Piper / XTTS)
3. If `library_search=True`: search `VOICE_MODEL_IDS` dict (hex ID → voice name, matched via `endswith`), then fall back to CSV `skyrim_voice_folder` match
4. If still empty: `dictionary_match()` using race+sex key (e.g., `NordRace` → `Male Nord` / `Female Nord`)
5. Default fallback: `Male Nord` / `Female Nord`

### Additional Features

- **Weather**: `data/Skyrim/skyrim_weather.csv` — ID-based lookup + classification fallback
- **Idles**: `data/Skyrim/skyrim_idles.csv` — emote action support with FormID resolution
- **Bio templates**: `BioTemplateManager` with tag-based expansion (`tags`, `tags_overwrite` columns)
- **Text limit**: 500 characters, truncated with `...`
- **Sentence filtering**: Blocks sentences containing `'assist'` (except first sentence)
- **Fast response mode**: Plays audio asynchronously + saves muted voiceline

### Voice Dictionaries

Three sets: `MALE_VOICE_MODELS_XVASYNTH`, `FEMALE_VOICE_MODELS_XVASYNTH`, `MALE_VOICE_MODELS_XTTS`, `FEMALE_VOICE_MODELS_XTTS`, `MALE_VOICE_MODELS_PIPERTTS`, `FEMALE_VOICE_MODELS_PIPERTTS`

Races: Argonian, Breton, DarkElf, HighElf, Imperial, Khajiit, Nord, Orc, Redguard, WoodElf

`VOICE_MODEL_IDS`: ~40 hex ID → voice name mappings (e.g., `'0002992B': 'Dragon'`)

## Fallout4 Implementation

**Extender**: F4SE  
**Character CSV**: `data/Fallout4/fallout4_characters.csv`  
**Voice folder column**: `fallout4_voice_folder`  
**XVASynth matches**: `data/Fallout4/FO4_Voice_folder_XVASynth_matches.csv`

### Voice Model Resolution (`find_best_voice_model`)

1. Parse voice model ID/name from parentheses/angle brackets
2. Select dictionary: XVASynth vs non-XVASynth (different naming conventions)
3. **Robot voice workarounds**: `RobotCompanionMaleDefault`/`RobotCompanionMaleProcessed` → `robotassaultron`; `SynthGen1Male02`/`SynthGen1Male03` → `gen1synth01`
4. If `library_search=True`: search `FO4_Voice_folder_and_models_df` by `voice_ID`, then `voice_file_name`, then CSV `fallout4_voice_folder`
5. If still empty: `dictionary_match()` using race+sex key
6. Default fallback: `maleboston` / `femaleboston`

### Additional Features

- **Text limit**: 448 bytes (UTF-8), with multi-byte character boundary handling
- **VR support**: `file_communication_compatibility` for Fallout 4 VR
- **FUZ files**: Copies `.fuz` lip sync files alongside audio
- **No weather**: `get_weather_description()` returns empty string
- **No sentence filtering**: `is_sentence_allowed()` always returns True

### Voice Dictionaries

Two sets: `MALE_VOICE_MODELS_XVASYNTH` / `FEMALE_VOICE_MODELS_XVASYNTH` and `MALE_VOICE_MODELS_NONXVASYNTH` / `FEMALE_VOICE_MODELS_NONXVASYNTH`

Races: Assaultron, DLC01RoboBrain, DLC02Handy, DLC02FeralGhoul, DLC03_SynthGen2, DLC03RoboBrain, EyeBot, Ghoul, FeralGhoul, FeralGhoulGlowing, Human, Protectron, SupermutantBehemoth, SuperMutant, SynthGen1, SynthGen2, TurretBubble, TurretTripod, TurretWorkshop

## Equipment System

`Equipment` (`src/games/equipment.py`) maps slot names to `EquipmentItem` objects:

- **Slots**: `body`, `head`, `hands`, `feet`, `amulet`, `righthand`, `lefthand`, `spells`
- **Description order**: body → head → hands → feet → amulet (armor), then weapons, then spells
- **Output format**: `"{name} wears {armor} and uses {weapons} and knows the spells {spells}."`

## External Character Info

`external_character_info` (`src/games/external_character_info.py`) is a DTO with:

- `name`, `is_generic_npc`, `bio`
- `ingame_voice_model`, `tts_voice_model`, `csv_in_game_voice_model`, `advanced_voice_model`
- `voice_accent`
- `llm_service`, `llm_model`, `tts_service` (per-character overrides)

## Character Data Model

`Character` (`src/character_manager.py`) properties:

- **IDs**: `base_id`, `ref_id` (with setters for dynamic updates)
- **Name**: `name` (with setter)
- **Gender**: `gender_raw` (0=male, 1=female), `gender` (readable), pronouns (`personal_pronoun_subject`, `personal_pronoun_object`, `possesive_pronoun`)
- **Race**: `race_raw` (raw game value), `race` (parsed via `utils.parse_race_name`)
- **State**: `is_player_character`, `is_in_combat`, `is_enemy`, `relationship_rank`, `is_generic_npc`
- **Voice**: `in_game_voice_model`, `tts_voice_model`, `csv_in_game_voice_model`, `advanced_voice_model`, `voice_accent`
- **Other**: `equipment`, `custom_character_values`, `llm_service`, `llm_model`, `tts_service`

Helper functions: `get_genders_text()`, `get_races_text()`, `get_genders_and_races_text()` — format character lists as natural language.

## Characters Manager

`Characters` (`src/characters_manager.py`) manages the NPC list:

- **Active characters**: `dict[name, Character]` — currently in conversation
- **All since start**: `dict[name, Character]` — includes NPCs who left
- **Participation log**: `list[(event, name, message_index)]` — ordered join/leave events
- **Nearby NPCs**: Lightweight `list[dict]` for NPCs near the conversation
- **Pending shares**: `list[(sharer, recipient, ref_id)]` — NPCs to receive conversation summary
- **Key methods**: `add_or_update_character()`, `remove_character()`, `get_character_by_name()`, `contains_multiple_npcs()`, `get_all_names_w_nearby()`

## GameStateManager

`GameStateManager` (`src/game_manager.py`) orchestrates conversations:

- **Lifecycle**: `start_conversation()` → `continue_conversation()` / `player_input()` → `end_conversation()`
- **Character loading** (`load_character`): Converts JSON → `Character`, resolves IDs (handles `FE` prefix for lite mods), loads external info via `game.load_external_character_info()`
- **Context updates** (`__update_context`): Processes actors, location, time, weather, nearby NPCs, config settings, custom values from JSON
- **Random LLM**: `_build_random_conversation_client()` — optionally creates a random LLM client per conversation
- **Voice preloading**: `__try_preload_voice_model()` — pre-loads voice when single NPC speaks first without narrator
- **Action handling**: Detects action keywords in player input, handles internal actions (listen, vision), dispatches action-only responses

## Character Override System

Override folders (JSON or CSV):

1. **Mod overrides**: `<mod_path>/<extender>/Plugins/MantellaSoftware/data/<game>/character_overrides/`
2. **Personal overrides**: `<save_folder>/data/<game>/character_overrides/`

JSON format: `{"name": "...", "base_id": "...", "race": "...", "voice_model": "...", ...}`  
Matching uses the same `_get_matching_df_rows_matcher` priority chain. Unmatched characters are added as new rows.

## Bio Template System

`BioTemplateManager` (`src/bio_template_manager.py`) — Skyrim-only:

- **3-tier cascade**: Base → mod overrides → personal overrides (highest priority)
- **Tag expansion**: `expand_bio_with_tags(bio, tags, tags_overwrite)` injects template text based on character tags
- **Enable/disable**: Controlled by `config.enable_character_tag_reading`

## Key Config Values

`game`, `game_path`, `mod_path`, `mod_path_base`, `save_folder`, `language`, `player_name`, `player_voice_model`, `voice_player_input`, `automatic_greeting`, `narration_handling`, `fast_response_mode`, `fast_response_mode_volume`, `save_audio_data_to_character_folder`, `enable_character_tag_reading`, `random_llm_enabled`, `random_llm_pool`, `remove_mei_folders`

## Common Patterns

- **Adding a new game**: Extend `Gameable`, implement all abstract methods, add voice dictionaries, add `GameEnum` entry, update `ConfigLoader` detection
- **Modifying voice resolution**: Edit `find_best_voice_model()` in the respective game class; coordinate with TTS specialist
- **Adding CSV columns**: Add to character CSV → ensure override system picks it up → update `load_external_character_info()` return
- **ID handling**: Always use `utils.convert_to_skyrim_hex_format()` for IDs from the game; strip `FE` prefix for lite mods
- **Text limits**: Skyrim = 500 chars, Fallout 4 = 448 bytes (UTF-8 aware)

## Tests

- `tests/test_game_manager.py` — conversation lifecycle, reload, JSON schema validation
- `tests/games/` — game-specific tests
- Tests use `@pytest.mark.requires_llm` for integration tests that need a live LLM
