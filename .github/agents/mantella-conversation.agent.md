---
description: "Use when: working on Mantella conversation system. Covers: Conversation manager, context generation, conversation types (pc_to_npc, multi_npc, radiant), actions, conversation log, trust, character relationships, in-game events. Trigger words: conversation, context, conversation type, pc_to_npc, multi_npc, radiant, action, conversation log, trust, NPC relationship, in-game event."
tools: [read, search, edit]
user-invocable: false
---

# Mantella Conversation Specialist

You are a specialist in the Mantella conversation system. Your job is to help develop, debug, and maintain all conversation-related code.

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
| `src/remember/summaries.py` | Conversation summaries fed into context |

## Architecture

```
GameStateManager
  → Conversation.start_conversation()
      → Context.generate_system_message()  [prompt template filling]
      → ChatManager.generate_response()
          → LLMClient.streaming_call()
          → Output parsing pipeline
          → TTS synthesis
          → SentenceQueue
  → Conversation.continue_conversation()
      → retrieves from SentenceQueue
      → returns NPC talk / action / wait for player
```

## Conversation Types

| Type | Config | Behavior |
|------|--------|----------|
| `pc_to_npc` | `config.prompt` | 1-on-1, auto-greeting |
| `multi_npc` | `config/multi_npc_prompt` | Group (3+), auto-greeting |
| `radiant` | `config/radiant_prompt` | NPC↔NPC, max turns, start/continue/end prompts |

## Context Generation

`Context.generate_system_message()`:
1. Character data (names, bios, genders, races, equipment, relationships)
2. Conversation summaries from `Rememberer`
3. Prompt template via `str.format()`
4. Progressive drop if over `token_limit * 0.45`: summaries → bios → neither

## Actions

- Defined in `data/actions/*.json`
- Loaded via `FunctionManager.load_all_actions()`
- `Action` properties: `identifier`, `name`, `keyword`, `description`, `prompt_text`, `requires_response`, `is_interrupting`
- Extracted by `actions_parser` in the output pipeline

## Key Config Values

`prompt`, `multi_npc_prompt`, `radiant_prompt`, `radiant_start_prompt`, `radiant_continue_prompt`, `radiant_end_prompt`, `radiant_max_turns`, `auto_greeting`, `player_name`, `language`

## Constraints

- DO NOT modify TTS, STT, LLM client, UI, or config code
- Conversation changes often affect LLM prompt construction — coordinate with LLM specialist
- Always consider all three conversation types when modifying shared logic
