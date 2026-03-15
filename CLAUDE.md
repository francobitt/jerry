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
├── .env             # OLLAMA_BASE_URL (not committed)
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
| `model` | string | Ollama model name |
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
| 6–7 | **`TASK`** / `SKILL` + `SKILL_ARGS` — set what the agent should do |
| 8 | `load_skill()` + `_resolve_task()` — skills loader |
| 9 | `ollama_chat()` + `check_ollama()` |
| 10 | `filter_tools()` + `to_ollama_tools()` |
| 11 | `extract_observation()` + `capture_screenshot()` + `_gif_frames` buffer |
| 12 | `make_run_dir()` + `save_artifacts()` + `save_gif()` |
| 13 | `confirm_action()` — human-in-the-loop widget |
| 14 | `load_session()` + `save_session()` |
| 15 | `maybe_compress()` + `call_tool_safely()` |
| 16 | `run_agent()` — ReAct loop |
| 17 | `start_scheduler()` + `stop_scheduler()` |
| 18 | `main()` + **Run** |

## Architecture notes
- `npx` is resolved at startup via `shutil.which` with fallback to common install paths; a clear error is raised if Node is not installed
- `env=dict(os.environ)` is passed explicitly to `StdioServerParameters` to prevent PATH stripping in Jupyter subprocess environments
- Ollama tool-role messages do not support the `images` field; image observations are attached as a follow-up `role: "user"` message instead
- `tool_call_id` is included in tool response messages when the model provides an `id` on the call
- The MCP session is opened once per `main()` call and shared across all ReAct steps
- GIF frames are collected in a module-level `_gif_frames` list; `main()` clears it at the start of each run to prevent cross-run contamination
- Human-in-the-loop uses `asyncio.Event` for non-blocking widget confirmation compatible with `nest_asyncio`
- Scheduled runs each get their own isolated browser session, run folder, and cleared frame buffer

## Version control
- Claude manages commits, tags, and `CHANGELOG.md` updates
- Auto-push hook configured in `.claude/settings.local.json`: pushes to GitHub after every `git commit`
- `.env` and `.claude/` are gitignored and never committed
- Tag format: `jerry<major>.<minor>` (e.g. `jerry0.2`)
