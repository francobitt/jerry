# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Jerry — Local Browser Automation Agent

## Project overview
Jerry is a local ReAct-style browser automation agent. It runs entirely on-device using an Ollama-hosted LLM and a Chromium browser controlled via the Playwright MCP server. All agent logic lives in a single Jupyter notebook (`agent.ipynb`).

## Stack
- **LLM**: Ollama (`qwen3-vl:30b` by default) — served at `OLLAMA_BASE_URL` from `.env`
- **Browser control**: `@playwright/mcp@latest` npm package, launched as a subprocess via `npx`
- **MCP client**: Python `mcp` SDK (`StdioServerParameters` + `stdio_client`)
- **Notebook runtime**: Jupyter with `nest_asyncio` for async compatibility
- **GIF replay**: `Pillow` — stitches captured frames into `replay.gif`
- **Scheduling**: `APScheduler` — cron-based recurring task execution
- **HITL widgets**: `ipywidgets` — confirmation prompts for dangerous actions

## File structure
```
Jerry/
├── agent.ipynb      # All agent logic — edit SYSTEM_PROMPT and TASK here
├── config.json      # Runtime configuration (see below)
├── skills/          # Reusable skill definitions (YAML)
│   ├── summarize_page.yaml
│   ├── login.yaml
│   └── search_and_extract.yaml
├── credentials/         # Gitignored — one YAML per credential set
│   └── <name>.yaml      # e.g. github.yaml: username/password/etc.
├── credentials.example/ # Committed template — copy to credentials/ and fill in
├── .env             # OLLAMA_BASE_URL / LMSTUDIO_BASE_URL (not committed)
├── .env.example     # Template for .env
├── requirements.txt # Python dependencies
├── runs/            # Created at runtime — one folder per run
│   └── <timestamp>/
│       ├── result.json   # Structured output from the LLM
│       ├── summary.txt   # Plain-text final answer
│       └── replay.gif    # Animated GIF of all screenshots
├── session.json     # Browser auth state (if session_file is configured)
├── CHANGELOG.md     # Version history
└── CLAUDE.md        # This file
```

## Configuration (`config.json`)

### Core
| Key | Type | Description |
|-----|------|-------------|
| `backend` | string | LLM provider: `"ollama"` (default) or `"lmstudio"` |
| `model` | string | Model name (passed to whichever backend is active) |
| `max_steps` | int | Hard cap on ReAct loop iterations |
| `headless` | bool | `true` = no browser window |
| `use_vision` | string | `"false"` / `"true"` / `"auto"` |
| `flash_mode` | bool | `true` = suppress `<think>` tokens (`think: false` in Ollama API) |

### Session persistence
| Key | Type | Description |
|-----|------|-------------|
| `session_file` | string\|null | Path to save/restore browser auth state; `null` = disabled |

### Artifacts & GIF replay
| Key | Type | Description |
|-----|------|-------------|
| `save_artifacts` | bool | Save `result.json` + `summary.txt` to `./runs/<timestamp>/` |
| `save_gif` | bool | Stitch captured screenshots into `replay.gif` |
| `gif_frame_ms` | int | Milliseconds per frame in the GIF |

### Human-in-the-loop
| Key | Type | Description |
|-----|------|-------------|
| `human_in_loop` | bool | Enable confirmation widget before dangerous actions |
| `confirm_keywords` | list | Action names/args containing these words trigger a pause |

### Retry & error recovery
| Key | Type | Description |
|-----|------|-------------|
| `retry_enabled` | bool | Retry failed tool calls |
| `max_retries` | int | Maximum retry attempts per tool call |
| `retry_delay_s` | float | Seconds between retries |

### Context compression
| Key | Type | Description |
|-----|------|-------------|
| `context_compress_after` | int | Compress history after this many turns; `0` = disabled |

### Loop detection
| Key | Type | Description |
|-----|------|-------------|
| `loop_detection` | bool | Detect and handle repeated action fingerprints |
| `loop_window` | int | Number of past step fingerprints to compare against (default `5`) |
| `max_loop_repeats` | int | Consecutive repeats before hard-stopping with a summary (default `3`) |

### Model timeout
| Key | Type | Description |
|-----|------|-------------|
| `model_timeout_s` | int | Seconds to wait for a model response before killing the run; `0` = disabled (default `120`) |

### Orchestrator
| Key | Type | Description |
|-----|------|-------------|
| `max_subagents` | int | Max sub-agents per `spawn_subagents` call (default `6`); hard cap on parallel browser instances |

### Scheduling
| Key | Type | Description |
|-----|------|-------------|
| `schedule` | string\|null | Cron expression for recurring runs (e.g. `"0 * * * *"`); `null` = run once |

### `use_vision` behaviour
- `"false"` — screenshot tools hidden from model entirely
- `"true"` — screenshot tool hidden; loop auto-captures after every action round and injects as a user message
- `"auto"` — `browser_take_screenshot` exposed as a tool; model calls it on demand

## Environment setup
```bash
brew install node          # Node.js required for npx / @playwright/mcp
pip install -r requirements.txt
cp .env.example .env       # then set OLLAMA_BASE_URL (default: http://localhost:11434)
ollama pull qwen3-vl:30b
```

There are no build, lint, or test commands — all logic lives in the notebook and is executed interactively.

## Running the agent
1. Open `agent.ipynb` in Jupyter
2. Edit `SYSTEM_PROMPT` (cell 5) and `TASK` (cell 7)
3. Adjust `config.json` as needed
4. Run all cells — the **Run Agent** cell (last) launches the browser and starts the loop

## Notebook cell map
| Cell | Purpose |
|------|---------|
| 1 | `%pip install` |
| 2 | Imports + `nest_asyncio.apply()` |
| 3 | Config & environment loading, `npx` resolution |
| 4–5 | **`SYSTEM_PROMPT`** — edit here |
| 6 | **`ORCHESTRATOR_SYSTEM_PROMPT`** — orchestrator decomposition instructions |
| 7–8 | **`TASK`** / `SKILL` + `SKILL_ARGS` / `ORCHESTRATOR_TASK` + `SUBAGENT_MODE` |
| 9 | `load_skill()` + `_resolve_task()` + `_expand_creds()` — skills & credentials loader |
| 10 | `ollama_chat()` + `check_ollama()` |
| 11 | `filter_tools()` + `to_ollama_tools()` |
| 12 | `extract_observation()` + `capture_screenshot()` + `_gif_frames` buffer |
| 13 | `make_run_dir()` + `save_artifacts()` + `save_gif()` |
| 14 | `confirm_action()` — human-in-the-loop widget |
| 15 | `load_session()` + `save_session()` |
| 16 | `maybe_compress()` + `call_tool_safely()` |
| 17 | `run_agent()` — ReAct loop |
| 18 | `_run_one_agent()` + `run_orchestrator()` — orchestrator + sub-agent runner |
| 19 | `start_scheduler()` + `stop_scheduler()` |
| 20 | `main()` + **Run** (dispatches to scheduler / orchestrator / single agent) |

## Architecture notes
- `npx` is resolved at startup via `shutil.which` with fallback to common install paths; a clear error is raised if Node is not installed
- `env=dict(os.environ)` is passed explicitly to `StdioServerParameters` to prevent PATH stripping in Jupyter subprocess environments
- Ollama tool-role messages do not support the `images` field; image observations are attached as a follow-up `role: "user"` message instead
- `tool_call_id` is included in tool response messages when the model provides an `id` on the call
- The MCP session is opened once per `main()` call and shared across all ReAct steps
- GIF frames are collected in a module-level `_gif_frames` list; `main()` clears it at the start of each run to prevent cross-run contamination; sub-agents each receive their own `frames: list` passed via `frame_buffer=` to avoid interleaving
- Human-in-the-loop uses `asyncio.Event` for non-blocking widget confirmation compatible with `nest_asyncio`
- Scheduled runs each get their own isolated browser session, run folder, and cleared frame buffer
- Orchestrator sub-agents each open their own `stdio_client` → `ClientSession` — fully isolated Chromium processes; `asyncio.gather` is used for parallel mode

## Credential store

Credentials live in `credentials/<name>.yaml` (gitignored). Each file is a flat key/value YAML:

```yaml
username: alice
password: s3cr3t
```

Reference individual values anywhere in `TASK` or `SKILL_ARGS` using the `creds:name.key` token syntax — tokens are expanded by `_expand_creds()` inside `_resolve_task()` before any text reaches the model.

```python
# With a skill
SKILL_ARGS = {
    "url":      "https://github.com/login",
    "username": "creds:github.username",
    "password": "creds:github.password",
}

# With a raw task
TASK = "Log in to https://github.com using creds:github.username and creds:github.password"
```

`credentials.example/` ships with the repo as a format reference; copy it to `credentials/` and fill in real values.

## Version control
- Claude manages commits, tags, and `CHANGELOG.md` updates
- Auto-push hook configured in `.claude/settings.local.json`: pushes to GitHub after every `git commit`
- `.env` and `.claude/` are gitignored and never committed
- Tag format: `jerry<major>.<minor>.<patch>` — **major** = architectural overhaul or breaking change, **minor** = new feature or significant capability, **patch** = small improvement or bug fix (e.g. `jerry1.0.0`, `jerry0.7.0`, `jerry0.6.1`)
