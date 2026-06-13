---
name: mantella-llm
description: "Domain knowledge for Mantella LLM subsystem. Covers: LLM clients (LLMClient, FunctionClient, ImageClient, SummaryLLMClient), AI services (OpenAI, OpenRouter, Claude), prompts, conversation types, message threads, actions/function calling, vision, summaries, token counting, model profiles, per-character overrides, output parsing. Load when working on LLM features."
---

# Mantella LLM Domain Knowledge

## LLM Client Hierarchy

```
AIClient (ABC) — src/llm/ai_client.py
  └─ ClientBase — src/llm/client_base.py
      ├─ LLMClient — src/llm/llm_client.py (main NPC response)
      │   ├─ ImageClient — src/llm/image_client.py (vision)
      │   └─ FunctionClient — src/llm/function_client.py (tool calling)
      ├─ SummaryLLMClient — src/llm/summary_client.py
      └─ LLMTestClient — src/llm/llm_test_client.py (mock)
```

## LLM Services

All use OpenAI Python SDK with compatible endpoints:
- OpenAI: `https://api.openai.com/v1`
- OpenRouter: `https://openrouter.ai/api/v1`
- NanoGPT: `https://nano-gpt.com/api/v1`
- KoboldCpp: `http://127.0.0.1:5001/v1`
- textgenwebui: `http://127.0.0.1:5000/v1`
- Custom URLs passed through

## API Key Resolution

1. `secret_keys.json` in mod folder
2. `secret_keys.json` in local folder
3. `GPT_SECRET_KEY.txt` in local folder

## Parameter Resolution

```
config.llm + config.llm_api + config.llm_params
  → ModelProfileManager.resolve_params()
      → Profile exists? Use profile params
      → Else? Use config llm_params
```

## Conversation Types

| Type | Config | Behavior |
|------|--------|----------|
| `pc_to_npc` | `config.prompt` | 1-on-1, auto-greeting |
| `multi_npc` | `config/multi_npc_prompt` | Group (3+), auto-greeting |
| `radiant` | `config/radiant_prompt` | NPC↔NPC, max turns |

## Prompt Construction

`Context.generate_system_message()`:
1. Gathers character data (names, bios, genders, races, equipment, relationships)
2. Gets conversation summaries from `Rememberer`
3. Fills template via `str.format()`
4. Progressively drops content if over `token_limit * 0.45`:
   - Try: bios + summaries → bios only → neither

## Message Types

- `SystemMessage`, `UserMessage`, `AssistantMessage`, `ImageMessage`, `ImageDescriptionMessage`, `ToolMessage`
- `message_thread` — ordered collection with truncation and cloning

## Output Parsing Pipeline

`ChatManager.process_response()` chains:
1. `change_character_parser` — character switches
2. `italics_parser` — italic markers
3. `narration_parser` — narration vs speech
4. `sentence_end_parser` — sentence boundaries
5. `actions_parser` — action keyword extraction
6. `sentence_length_parser` — word count limits
7. `max_count_sentences_parser` — max sentence count

## Per-Character LLM Overrides

- Config: `allow_per_character_llm_overrides`
- CSV columns: `llm_service`, `llm_model`
- `ChatManager._get_per_character_client()` caches by `{ref_id}_{endpoint}_{model}`

## Actions / Function Calling

- Actions defined in `data/actions/*.json`
- Loaded via `FunctionManager.load_all_actions()`
- `FunctionClient.check_for_actions()` detects action keywords
- Action keywords extracted by `actions_parser`

## Token Counting

- Uses `tiktoken`
- Context limit: `token_limit * TOKEN_LIMIT_PERCENT (0.45)`
- `ClientBase.__num_tokens_from_messages()` for counting

## Key Config Values

`llm`, `llm_api`, `llm_params`, `custom_token_count`, `claude_prompt_caching_enabled`, `apply_model_profiles`, `vision_enabled`, `custom_vision_model`, `vision_llm_api`, `vision_llm`, `vision_llm_params`, `advanced_actions_enabled`, `custom_function_model`, `function_llm_api`, `function_llm`, `function_llm_params`, `summary_llm_enabled`, `summary_llm_api`, `summary_llm`, `summary_llm_params`, `allow_per_character_llm_overrides`, `random_llm_pool`

## Tests

- `tests/llm/` — various LLM tests
- `LLMTestClient` provides mock responses for testing
