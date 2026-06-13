---
name: mantella-remember
description: "Domain knowledge for Mantella memory/remember system. Covers: conversation summaries, remembering, conversation history, long-term memory. Load when working on memory features."
---

# Mantella Remember Domain Knowledge

## Architecture

The remember system manages conversation memory through summaries that are injected into future conversation prompts.

```
Conversation.__save_conversation()
  → Rememberer.save_conversation_state()
      → get_threads_for_summarization() — per-NPC message filtering
      → group_shared_threads() — dedup identical NPC experiences
      → __create_new_conversation_summary() → SummaryLLMClient
      → __append_new_conversation_summary() → writes to disk
      → (if token limit exceeded) resummarize into new file

Context.generate_system_message()
  → Rememberer.get_prompt_text()
      → reads latest summary files from disk
      → injects into system prompt
```

## Key Files

| File | Purpose |
|------|---------|
| `src/remember/remembering.py` | `Remembering` ABC — interface for all remember implementations |
| `src/remember/summaries.py` | `Summaries` — file-based summary storage, the primary `Remembering` implementation |
| `src/llm/summary_client.py` | `SummaryLLMClient` — dedicated LLM client for summary generation |
| `src/conversation/conversation_log.py` | `conversation_log` — raw conversation history save/load (JSON) |

## Remembering ABC (`src/remember/remembering.py`)

Abstract base class defining the remember interface:

- **`get_prompt_text(characters, world_id) -> str`** — Loads stored summaries for given NPCs and returns them as prompt text for the LLM system message.
- **`save_conversation_state(messages, npcs_to_summarize, npcs_in_conversation, world_id, ...)`** — Generates and persists conversation summaries for the given NPCs.

## Summaries Class (`src/remember/summaries.py`)

The primary `Remembering` implementation. Stores summaries as `.txt` files on disk.

### Constructor

```python
Summaries(game, config, client, language_name, summary_client=None, summary_limit_pct=0.3)
```

- `summary_client` — Optional `SummaryLLMClient`. Falls back to the main `LLMClient` if not provided.
- `summary_limit_pct` — Fraction of the LLM token limit before triggering resummarization (default 30%).

### Summary Generation Flow (`save_conversation_state`)

1. **Per-NPC thread extraction** (`get_threads_for_summarization`): Uses the `Characters` participation log (join/leave events) to build per-NPC message threads. Each NPC only gets the messages they actually heard. NPCs who leave and rejoin only get messages from their **latest interval**.

2. **Thread deduplication** (`group_shared_threads`): Groups NPCs with identical message sequences (by formatted content) to share a single LLM summarization call.

3. **Minimum message threshold**: Radiant conversations bypass the threshold. Reload saves use `min_messages=2`. Normal saves use `min_messages=5`.

4. **Summary creation** (`__create_new_conversation_summary`): Builds a prompt from `memory_prompt` config, NPC bios, genders, races, and existing summaries. Calls `summarize_conversation()` via the summary client. Optionally prepends a game timestamp if `memory_prompt_datetime_prefix` is enabled.

5. **File writing** (`__append_new_conversation_summary`): Appends the summary to the latest summary file. If the token limit is exceeded, triggers resummarization: the existing summaries are condensed into a single summary written to a new file (incremented number).

6. **Pending shares**: If `pending_shares` is provided, writes prefixed summaries to recipient NPC folders (e.g., "Guard shared with Lydia a conversation with...").

### Summary File Layout

```
<conversation_folder_path>/<world_id>/<NPC Name> - <ref_id>/<NPC Name>_<N>.txt
```

- Prioritizes `name - ref_id` folders; falls back to legacy `name` folders.
- File numbering: `Guard_summary_1.txt`, `Guard_summary_2.txt`, etc.
- New resummarization creates a new file with an incremented number.

### Prompt Text Retrieval (`get_prompt_text`)

- Reads the latest summary file for each NPC.
- Deduplicates lines across reads.
- For multi-NPC conversations, wraps each NPC's summary with `[This is the beginning/end of <name>'s memory]`.
- Returns all summaries prefixed with `"Below is a summary of past events:\n"`.

### Text Post-Processing (`summarize_conversation`)

After the LLM returns a summary, replacements are applied:
- `The assistant` / `the assistant` / `an assistant` / `an AI assistant` → `Someone`
- `The user` / `the user` → `The player` / `the player`

### Timestamp Formatting (`__format_timestamp`)

Converts game time (days as float) to `[Day X, Y in the evening]` format using 12-hour time and time-of-day groups.

## SummaryLLMClient (`src/llm/summary_client.py`)

A dedicated LLM client extending `ClientBase` for summary generation.

```python
class SummaryLLMClient(ClientBase):
    def __init__(self, config):
        profile_manager = get_profile_manager()
        summary_llm_params = profile_manager.resolve_params(
            service=config.summary_llm_api,
            model=config.summary_llm,
            fallback_params=config.summary_llm_params,
            apply_profile=config.apply_model_profiles,
            log_context="SummaryLLMClient",
        )
        super().__init__(config.summary_llm_api, config.summary_llm,
                         summary_llm_params, config.summary_custom_token_count)
```

- Uses `ModelProfileManager` to resolve parameters, supporting per-model profiles.
- Falls back to `summary_llm_params` from config if no profile matches.
- `summary_custom_token_count` (default 4096) is the fallback token limit when the model's limit is unknown.

## Conversation Log (`src/conversation/conversation_log.py`)

Handles raw conversation history (JSON), separate from summaries:

- **`save_conversation_log(character, messages, world_id)`** — Appends new messages to the character's conversation history JSON file.
- **`load_conversation_log(character, world_id)`** — Loads and flattens all previous conversations for a character.
- **`get_conversation_log_length(character, world_id)`** — Returns total message count across all saved conversations.

File path: `<game_path>/<world_id>/<NPC Name> - <ref_id>/<NPC Name>.json`

## Key Config Values

| Config Key | Type | Description |
|---|---|---|
| `summary_llm_enabled` | bool | Enable/disable summary generation |
| `summary_llm_api` | string | API service for summaries (e.g., "OpenAI", "OpenRouter") |
| `summary_llm` | string | Model name for summaries |
| `summary_llm_params` | dict | JSON parameters for the summary LLM (temperature, max_tokens, etc.) |
| `summary_custom_token_count` | int | Fallback token limit for unknown summary models (default 4096) |
| `memory_prompt` | string | Prompt template for generating new summaries |
| `resummarize_prompt` | string | Prompt template for condensing existing summaries when token limit is reached |
| `memory_prompt_datetime_prefix` | bool | Whether to prepend game timestamps to summaries |
| `apply_model_profiles` | bool | Whether to use model profiles for summary LLM parameter resolution |

## Integration Points

- **`Context.__init__`** receives a `Remembering` instance and calls `get_prompt_text()` during system message generation.
- **`Conversation.__save_conversation()`** calls `Rememberer.save_conversation_state()` with the full message thread, NPCs to summarize, participation data, and optional pending shares.
- `Summaries` can use either a dedicated `SummaryLLMClient` or fall back to the main `LLMClient`.

## Tests

- `tests/remember/test_summaries.py` — Tests for per-NPC thread extraction, multi-NPC join/leave scenarios, NPC rejoin behavior, shared thread grouping, and summary file management.
