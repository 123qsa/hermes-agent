# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup
```bash
./setup-hermes.sh              # One-shot dev setup (uv, venv, install, symlink)
```

### Testing
**Always use `scripts/run_tests.sh`** — never call `pytest` directly. The wrapper enforces hermetic CI-parity (unsets credential env vars, TZ=UTC, LANG=C.UTF-8, 4 xdist workers). Direct pytest on a 16+ core machine diverges from CI and has caused multiple "works locally, fails in CI" incidents.
```bash
scripts/run_tests.sh                              # full suite
scripts/run_tests.sh tests/agent/                 # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
scripts/run_tests.sh -v --tb=long                 # pass-through pytest flags
```

### Running the agent
```bash
source venv/bin/activate
hermes                           # interactive CLI
hermes --tui                     # Ink/React TUI
hermes gateway start             # messaging gateway
hermes doctor                    # diagnostics
```

### TUI (ui-tui/)
```bash
cd ui-tui
npm install                      # first time
npm run dev                      # watch mode
npm run build                    # production build
npm run type-check               # tsc --noEmit
npm run lint                     # eslint
npm run test                     # vitest
```

### Web UI (web/)
```bash
cd web
npm install
npm run dev                      # vite dev server
npm run build                    # production build
npm run lint                     # eslint
```

### Skills Hub web (website/)
```bash
cd website
npm install
npm run dev                      # vite dev server
```

## Architecture

### Core Entry Points
- `run_agent.py` — `AIAgent` class, the core synchronous conversation loop (`run_conversation()`). Handles tool dispatch, session persistence, context compression.
- `cli.py` — `HermesCLI` class, interactive prompt_toolkit CLI orchestrator.
- `hermes_cli/main.py` — Entry point for all `hermes` subcommands.
- `gateway/run.py` — `GatewayRunner`, messaging platform lifecycle and message routing.
- `model_tools.py` — Thin orchestration layer over `tools/registry.py`; triggers tool discovery.
- `toolsets.py` — Tool groupings and platform presets.

### Self-Registering Tools
Tools auto-discover via `registry.register()` at import time. Any `tools/*.py` file with a top-level `registry.register()` call is imported automatically by `model_tools.py` — no manual import list.
```
tools/registry.py  (no deps — imported by all tool files)
       ↑
tools/*.py  (each calls registry.register() at import time)
       ↑
model_tools.py  (imports tools/registry + triggers tool discovery)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### TUI Process Model (`ui-tui/` + `tui_gateway/`)
```
hermes --tui
  └─ Node (Ink/React)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                      └─ AIAgent + tools + sessions
       └─ renders transcript, composer, prompts, activity
```
TypeScript owns the screen. Python owns sessions, tools, model calls, and slash command logic. Transport is newline-delimited JSON-RPC over stdio.

### Session Persistence
All conversations stored in SQLite (`hermes_state.py`) with FTS5 full-text search and unique session titles. JSON logs go to `~/.hermes/sessions/`.

## Important Rules

### Prompt Caching Must Not Break
Hermes ensures caching remains valid throughout a conversation. **Do NOT implement changes that would:**
- Alter past context mid-conversation
- Change toolsets mid-conversation
- Reload memories or rebuild system prompts mid-conversation

The ONLY time we alter context is during context compression.

### Never Hardcode `~/.hermes` Paths
Use `get_hermes_home()` from `hermes_constants` for code paths. Use `display_hermes_home()` for user-facing print/log messages. Hardcoding `~/.hermes` breaks profiles — each profile has its own `HERMES_HOME` directory. This was the source of 5 bugs fixed in PR #3575.

### Tests Must Not Write to Real `~/.hermes/`
The `_isolate_hermes_home` autouse fixture in `tests/conftest.py` redirects `HERMES_HOME` to a temp dir. When testing profile features, also mock `Path.home()` so that `_get_profiles_root()` resolves within the temp dir. See `tests/hermes_cli/test_profiles.py` for the pattern.

### Cross-Platform Compatibility
- `termios` and `fcntl` are Unix-only. Always catch both `ImportError` and `NotImplementedError`.
- Windows may save `.env` files in `cp1252`. Handle `UnicodeDecodeError` with fallback encoding.
- `os.setsid()`, `os.killpg()`, and signal handling differ on Windows. Use `platform.system()` checks.
- If changing `scripts/install.sh`, check if `scripts/install.ps1` also needs updating.

### Tool Schemas Must Not Hardcode Cross-Tool References
Tool schema descriptions must not mention tools from other toolsets by name (e.g., `browser_navigate` saying "prefer web_search"). Those tools may be unavailable, causing the model to hallucinate calls. If a cross-reference is needed, add it dynamically in `get_tool_definitions()` in `model_tools.py`.

### Do Not Use `simple_term_menu` for Interactive Menus
Rendering bugs in tmux/iTerm2 — ghosting on scroll. Use `curses` (stdlib) instead. See `hermes_cli/tools_config.py` for the pattern.

### Do Not Use `\033[K` (ANSI Erase-to-EOL) in Spinner/Display Code
Leaks as literal `?[K` text under `prompt_toolkit`'s `patch_stdout`. Use space-padding: `f"\r{line}{' ' * pad}"`.

## Adding a Tool
Requires changes in **2 files**:
1. **Create `tools/your_tool.py`** with a top-level `registry.register()` call. All handlers MUST return a JSON string.
2. **Add to `toolsets.py`** — either `_HERMES_CORE_TOOLS` or a new toolset.

## Adding a Skill
Bundled skills live in `skills/`, official optional skills in `optional-skills/`. Each skill directory contains a `SKILL.md` with YAML frontmatter (`name`, `description`, `version`, `platforms`, `required_environment_variables`, etc.). Skills are instructions injected into the system prompt — no Python code required.

## Adding a Slash Command
1. Add a `CommandDef` entry to `COMMAND_REGISTRY` in `hermes_cli/commands.py`.
2. Add handler in `HermesCLI.process_command()` in `cli.py`.
3. If gateway-available, add handler in `gateway/run.py`.

`CommandDef` fields: `name`, `description`, `category`, `aliases`, `args_hint`, `cli_only`, `gateway_only`, `gateway_config_gate`. Adding an alias only requires adding it to the `aliases` tuple — dispatch, help, Telegram menu, Slack mapping, and autocomplete all update automatically.

## Configuration
- `DEFAULT_CONFIG` in `hermes_cli/config.py` — default config.yaml values.
- `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` — `.env` variable metadata.
- Bump `_config_version` in `config.py` to trigger migration for existing users.

## Commit Style
Conventional Commits: `type(scope): description`. Types: `fix`, `feat`, `docs`, `test`, `refactor`, `chore`. Scopes: `cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `security`, etc.
