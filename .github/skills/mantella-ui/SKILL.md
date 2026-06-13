---
name: mantella-ui
description: "Domain knowledge for Mantella UI and HTTP server subsystem. Covers: Gradio UI, settings page, FastAPI, uvicorn, HTTP routes, config editor, model profiles UI, file communication, browser auto-launch. Load when working on UI features."
---

# Mantella UI Domain Knowledge

## Server Architecture

```
main.py
  → http_server() — src/http/http_server.py (FastAPI wrapper)
  → routes: [mantella_route, StartUI]
  → http_server.start(port, routes)
      → _setup_routes() → each route.add_route_to_server(app)
      → uvicorn.run(app, port=port)
```

## HTTP Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/mantella` | POST | Game conversation (start/continue/input/end) |
| `/ui` | GET | Gradio settings page |
| `/favicon.ico` | GET | Icon |

## Request Types (POST /mantella)

- `mantella_initialize` → init completed
- `mantella_start_conversation` → `game.start_conversation()`
- `mantella_continue_conversation` → `game.continue_conversation()`
- `mantella_player_input` → `game.player_input()`
- `mantella_end_conversation` → `game.end_conversation()`

## Routeable Interface

All routes extend `routeable` (`src/http/routes/routeable.py`):
- `add_route_to_server(app: FastAPI)` — register endpoints
- `_setup_route()` — lazy initialization
- `_can_route_be_used()` — checks config validity, re-initializes if changed

## Gradio UI

`StartUI.create_main_block()` (`src/ui/start_ui.py`):
- Theme: `gr.themes.Soft` (green, zinc, Montserrat)
- Custom CSS: `src/ui/style.css`
- Settings tab: iterates `config.definitions.base_groups`
- Footer: Installation Guide link
- Placeholder tabs: "Chat with NPCs", "NPC editor" (not wired)

## Settings UI Constructor

`SettingsUIConstructor` (`src/ui/settings_ui_constructor.py`) — `ConfigValueVisitor`:
- Bool → `gr.Checkbox`
- Int/Float → `gr.Number`
- String → `gr.Textbox`
- Selection → `gr.Dropdown`
- Path → `gr.Textbox` + file picker
- Multi-selection → `gr.CheckboxGroup`
- Group → `gr.Tab`
- Advanced → collapsible `gr.Accordion`
- Share-row → paired side-by-side

## Config ↔ UI Binding

- `ConfigValue.value` setter → `_on_value_change_callback` → `ConfigLoader` → `config.ini`
- `routeable._can_route_be_used()` → re-initializes route on config change

## Model Profiles UI

`ProfileUIHandler` (`src/ui/profile_ui_handler.py`):
- Save/Delete buttons for profile parameters
- Auto-load on model dropdown change
- Bridges to `ModelProfileManager` → `model_profiles.json`

## File Communication

`file_communication_compatibility` (`src/http/file_communication_compatibility.py`):
- Polls `_mantella_communication.txt` for JSON requests
- Forwards to `POST /mantella`, writes response back
- Enables game mods without HTTP capability

## UI Groups (11 tabs)

Game, LLM, TTS, STT, Vision, Actions, Language, Prompts, Startup, Profiles, Other

## Key Config Values

`port` (4999), `auto_launch_ui` (True), `play_startup_sound` (True), `show_http_debug_messages` (False), `advanced_logs` (False)

## Tests

- `tests/ui/test_start_ui.py` — single initialization test
