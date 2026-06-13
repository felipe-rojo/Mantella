---
name: mantella-conversation
description: "Domain knowledge for Mantella conversation system. Covers: Conversation orchestrator, context/prompt generation, conversation types (pc_to_npc, multi_npc, radiant), actions, sentence flow, output parsing pipeline, per-character overrides, trust/relationships, conversation summaries, key config values. Load when working on conversation features."
---

# Mantella Conversation Domain Knowledge

## Architecture

```
GameStateManager
  → Conversation.start_conversation()
      → conversation_type.generate_prompt()  [builds system message]
      → ChatManager.generate_response()
          → LLMClient.streaming_call()
          → sentence_accumulator (buffers tokens)
          → output_parser chain (sentence splitting)
          → TTS synthesis (per-character override)
          → SentenceQueue
  → Conversation.continue_conversation()
      → retrieve_sentence_from_queue()  [blocking get]
      → returns NPC talk / action / wait for player
  → Conversation.process_player_input()
      → mic input / text input
      → update_game_events()
      → __start_generating_npc_sentences()
```

## Key Files

| File | Purpose |
|------|---------|
| `src/conversation/conversation.py` | `Conversation` — main orchestrator, start/continue/end flow |
| `src/conversation/context.py` | `Context` — prompt generation, character data, token management |
| `src/conversation/conversation_type.py` | `pc_to_npc`, `multi_npc`, `radiant` conversation types |
| `src/conversation/action.py` | `Action` — in-game action representation |
| `src/conversation/conversation_log.py` | Conversation history save/load (JSON per NPC per world) |
| `src/output_manager.py` | `ChatManager` — orchestrates LLM + TTS, per-character overrides |
| `src/llm/sentence_queue.py` | `SentenceQueue` — thread-safe LLM→conversation bridge |
| `src/remember/summaries.py` | `Summaries` — conversation summaries fed into context |
| `src/actions/function_manager.py` | `FunctionManager` — loads & parses action definitions |
| `src/http/communication_constants.py` | Constants for HTTP communication with game |

## Conversation Types

All extend `conversation_type` ABC (`src/conversation/conversation_type.py`).

| Type | Config | Behavior |
|------|--------|----------|
| `pc_to_npc` | `config.prompt` | 1-on-1, auto-greeting via `get_user_message()` |
| `multi_npc` | `config.multi_npc_prompt` | Group (3+ NPCs), auto-greeting |
| `radiant` | `config.radiant_prompt` | NPC↔NPC, no player, max turns, start/continue/end prompts |

### Type Selection Logic

`Conversation.__update_conversation_type()` selects based on actor count:
- No player character → `radiant`
- 3+ active characters → `multi_npc`
- Otherwise → `pc_to_npc`

Type can change mid-conversation when actors are added/removed (`Context.add_or_update_characters()` → `have_actors_changed` flag).

### Radiant Turn Structure

Message pattern: `[system] [start_prompt] [llm_1] [continue] [llm_2] ... [end_prompt] [llm_final]`
- `radiant.get_user_message()` returns `radiant_start_prompt`, `radiant_continue_prompt`, or `radiant_end_prompt` based on turn count
- `radiant.should_end()` returns `True` when `len(messages) // 2 >= radiant_max_turns`

## Context / Prompt Generation

`Context.generate_system_message(prompt, actions)` fills template variables:

### Template Variables

| Variable | Source |
|----------|--------|
| `player_name` | Player character name |
| `player_description` | From game or config |
| `player_equipment` | Equipment descriptions |
| `player_gender`, `player_race` | Character properties |
| `name` | Last added NPC name |
| `names` | All NPC names (no player) |
| `names_w_player` | All names including player |
| `bio` / `bios` | NPC bios |
| `trust` | Relationship descriptions (e.g. "a friend to Lydia") |
| `gender`, `race` | Last added NPC |
| `genders`, `races`, `genders_and_races` | All NPCs |
| `equipment` | All NPC equipment descriptions |
| `location` | Current location |
| `weather` | Current weather |
| `time` | In-game hour (12h format) |
| `time_group` | "morning", "afternoon", etc. |
| `current_day` | Day number from game_days |
| `language` | Language setting |
| `conversation_summary` / `conversation_summaries` | From `Rememberer` |
| `actions` | Action prompt texts |

### Progressive Token Drop

If prompt exceeds `token_limit * 0.45` tokens, content is dropped in order:
1. Drop conversation summaries (keep bios)
2. Drop bios too (keep neither)
3. Log warning if both dropped

### Trust Calculation

`Context.__get_trust(npc)` uses `conversation_log.get_conversation_log_length()`:
- `relationship_rank == 0`: stranger → acquaintance → friend → close friend (based on log length)
- `relationship_rank == 4`: lover
- `relationship_rank > 0`: friend
- `relationship_rank < 0`: enemy

**Known bug**: Trust includes radiant conversation counts, inflating the value.

## Actions

### Action Definition

Defined in `data/actions/*.json`. Key properties:
- `identifier`: Internal ID (e.g. `mantella_npc_offended`)
- `name`: Display name
- `key`: Keyword for LLM to trigger (e.g. `Attack`)
- `prompt`: Instruction text with `{key}` placeholder
- `parameters`: JSON schema with `scope` for entity resolution
- `requires_response`: Whether game must respond before continuing
- `is_interrupting`: Stops LLM generation after this sentence
- `one-on-one`, `multi-npc`, `radiant`: Which conversation types can use this action

### Action Loading

`FunctionManager.load_all_actions()` reads JSON files into `_actions` dict.

### Action Parsing

`actions_parser` in the output pipeline detects `keyword:` in text, strips it, and appends `{'identifier': action.identifier}` to the sentence's actions list.

### Special System Actions (communication_constants)

- `ACTION_RELOADCONVERSATION` — triggers conversation reload
- `ACTION_ENDCONVERSATION` — triggers conversation end
- `ACTION_REMOVECHARACTER` — removes NPC from conversation

### Advanced Actions (Tool Use)

When `config.advanced_actions_enabled`, actions are exposed as LLM tools via `FunctionManager.generate_context_aware_tools()`. The LLM can call these as function calls instead of text keywords. Special tool identifiers:
- `mantella_npc_vision` — enables vision for next LLM call
- `mantella_npc_listen` — triggers extended STT pause
- `mantella_end_conversation` — ends conversation via tool call

## Sentence Flow

### Generation

1. `Conversation.__start_generating_npc_sentences()` spawns a `Thread`
2. `ChatManager.generate_response()` → `process_response()` (async)
3. LLM streams tokens → `sentence_accumulator` buffers text
4. Output parser chain splits into sentences
5. Each sentence → `ChatManager.generate_sentence()` → TTS → `SentenceQueue`

### Output Parser Chain (in order)

1. **`change_character_parser`** — detects `CharacterName:` prefix, switches speaker
2. **`italics_parser`** — detects `*text*` for narration vs speech
3. **`narration_parser`** — handles narration markers (if not deactivated)
4. **`sentence_end_parser`** — splits at sentence boundaries (`.`, `?`, `!`, etc.)
5. **`actions_parser`** — detects action keywords
6. **`sentence_length_parser`** — enforces max word count per sentence
7. **`max_count_sentences_parser`** — limits sentences per response

### Retrieval

`Conversation.retrieve_sentence_from_queue()` — blocking get from `SentenceQueue`. If a sentence has text, waits for previous audio to finish (`last_sentence_audio_length`) before returning, giving player time to interrupt.

### Sentence Queue Behavior

- `SentenceQueue` is thread-safe with separate get/put locks
- `is_more_to_come` flag: `get_next_sentence()` blocks until queue has items OR this flag is `False`
- `put_at_front()` — used for reload/end sentences that must go to the front
- Final empty sentence is always queued to unblock the game

## Per-Character Overrides

### LLM Overrides

`ChatManager._get_per_character_client(character)`:
- Enabled by `config.allow_per_character_llm_overrides`
- Uses `character.llm_service` and `character.llm_model` from CSV
- Caches clients by `ref_id + service + model` key
- Falls back to default client on failure

### TTS Overrides

`ChatManager.generate_sentence()`:
- Enabled by `config.allow_per_character_tts_overrides`
- Uses `character.tts_service` from CSV
- `_get_or_create_tts()` caches per-service TTS instances
- Falls back to default TTS on failure

## Conversation Summaries

`Summaries` (`src/remember/summaries.py`) extends `Remembering`:
- Saves summaries to `<game_path>/<world_id>/<npc_name>/summary.txt`
- `get_prompt_text()` returns summary text for context injection
- `save_conversation_state()` called on conversation end and reload
- Summary generation uses a separate `SummaryLLMClient` if configured
- Controlled by `config.conversation_summary_enabled`

## Conversation Log

`conversation_log` (`src/conversation/conversation_log.py`):
- Saves to `<game_path>/<world_id>/<npc_name>/<npc_name>.json`
- For generic NPCs (e.g. "Whiterun Guard"), appends `ref_id`: `<name> - <ref_id>/<name>.json`
- Stores full message arrays as JSON
- `get_conversation_log_length()` returns total message count (used for trust)

## In-Game Events

`Context.update_context()` generates events when:
- Location changes → "The location is now {location}."
- Time changes → "The time is {hour} {group}." or "The conversation now takes place {group}."
- Weather changes → weather text
- NPC combat status → "{name} is now in combat!" / "no longer in combat."
- NPC enemy status → "{name} is attacking {player}." / "no longer attacking."
- NPC relationship rank → "{player} is now {trust} to {name}."
- Nearby NPCs change → "Characters nearby (from nearest to furthest): ..."
- Vision hints → "Characters currently in view: ..."

Events are added to player messages via `Conversation.update_game_events()`.

## Silence Auto-Response

When `config.silence_auto_response_enabled`:
- If player doesn't speak within `silence_auto_response_timeout` seconds, auto-fills with `silence_auto_response_message`
- Tracks consecutive count; disables after `silence_auto_response_max_count`
- Resets counter when player speaks

## Key Config Values

| Config | Purpose |
|--------|---------|
| `prompt` | pc_to_npc system prompt template |
| `multi_npc_prompt` | multi_npc system prompt template |
| `radiant_prompt` | radiant system prompt template |
| `radiant_start_prompt` | First turn prompt for radiant |
| `radiant_continue_prompt` | Mid-turn prompt for radiant |
| `radiant_end_prompt` | Final turn prompt for radiant |
| `radiant_max_turns` | Max LLM exchanges in radiant |
| `automatic_greeting` | Auto-greeting on conversation start |
| `end_conversation_keyword` | Keywords to end conversation |
| `goodbye_npc_response` | Goodbye text on conversation end |
| `collecting_thoughts_npc_response` | Text during conversation reload |
| `max_response_sentences_single` | Max sentences per response (1-on-1) |
| `max_response_sentences_multi` | Max sentences per response (multi) |
| `number_words_tts` | Max words per TTS sentence |
| `narration_handling` | How to handle narration (deactivate / use_narrator / cut) |
| `narration_start_indicators` | Characters marking narration start |
| `narration_end_indicators` | Characters marking narration end |
| `speech_start_indicators` | Characters marking speech start |
| `speech_end_indicators` | Characters marking speech end |
| `advanced_actions_enabled` | Use LLM tool calling for actions |
| `allow_interruption` | Allow mic interruption |
| `silence_auto_response_enabled` | Auto-respond on silence |
| `silence_auto_response_timeout` | Seconds before auto-respond |
| `silence_auto_response_message` | Auto-response text |
| `silence_auto_response_max_count` | Max consecutive auto-responses |
| `events_refresh_time` | Seconds before events need updating |
| `max_count_events` | Max events per message |
| `allow_per_character_llm_overrides` | Enable per-character LLM |
| `allow_per_character_tts_overrides` | Enable per-character TTS |
| `voice_player_input` | Synthesize player input |
| `conversation_summary_enabled` | Generate conversation summaries |
| `hourly_time` | Use exact hour vs time group |
| `wait_time_buffer` | Extra wait between sentences |
| `player_character_description` | Default player description |
| `language` | Language dict for greetings etc. |

## Common Patterns

### Starting a Conversation

1. Game sends `start_conversation` with NPC data
2. `Conversation.__init__()` creates `SentenceQueue`, determines `conversation_type`
3. `start_conversation()` calls `conversation_type.get_user_message()` for auto-greeting
4. If greeting exists, adds as `UserMessage` and starts generation
5. Returns `KEY_REPLYTYPE_NPCTALK` (greeting sent) or `KEY_REPLYTYPE_PLAYERTALK` (wait for player)

### Continuing a Conversation

1. `continue_conversation()` checks: ended? → too long? → player interrupting? → next sentence?
2. If sentence in queue with text → wait for previous audio → return `KEY_REPLYTYPE_NPCTALK`
3. If sentence with only actions → return `KEY_REPLYTYPE_NPCACTION`
4. If no sentence → check `should_end()` → get auto user message → or return `KEY_REPLYTYPE_PLAYERTALK`

### Ending a Conversation

- Player says goodbye keyword → `initiate_end_sequence()` → goodbye sentence + `ACTION_ENDCONVERSATION`
- LLM calls `mantella_end_conversation` tool → same flow
- `radiant.should_end()` returns true after max turns
- `end()` saves conversation log and summaries

### Conversation Reload

When messages exceed `token_limit * 0.9`:
1. `__initiate_reload_conversation()` → "gather thoughts" sentence + `ACTION_RELOADCONVERSATION`
2. Game triggers reload → `reload_conversation()` → save state → rebuild message thread

### Action Flow with Game Response

1. LLM triggers action (text keyword or tool call)
2. Action sentence sent to game via `KEY_REPLYTYPE_NPCACTION`
3. If `requires_response`, sets `interrupting_action = True` → stops LLM generation
4. Game performs action, sends result as in-game events
5. `resume_after_interrupting_action()` injects synthetic user message with events
6. Restarts generation (without tool use to prevent loops)

## Tests

- `tests/conftest.py` — shared fixtures (`MockAIClient`, test helpers)
- `tests/conversation/` — conversation-specific tests
- `tests/test_output_manager.py` — ChatManager tests
- `tests/test_game_manager.py` — integration tests

Test markers: `@pytest.mark.requires_audio`, `@pytest.mark.requires_external_exe`, `@pytest.mark.requires_llm`

## Constraints

- DO NOT modify TTS, STT, LLM client, UI, or config code
- Conversation changes often affect LLM prompt construction — coordinate with LLM specialist
- Always consider all three conversation types when modifying shared logic
- `Context.TOKEN_LIMIT_PERCENT = 0.45` (prompt) vs `Conversation.TOKEN_LIMIT_PERCENT = 0.9` (messages)
