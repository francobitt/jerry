# Jerry

A local ReAct-style browser automation agent. Runs entirely on-device using an Ollama-hosted LLM and a Chromium browser controlled via the Playwright MCP server.

## Stack

- **LLM**: Ollama (`qwen3-vl:30b` by default)
- **Browser control**: `@playwright/mcp@latest` via `npx`
- **MCP client**: Python `mcp` SDK
- **Notebook runtime**: Jupyter + `nest_asyncio`

## Setup

```bash
brew install node
pip install -r requirements.txt
cp .env.example .env   # set OLLAMA_BASE_URL
ollama pull qwen3-vl:30b
```

## Running

1. Open `agent.ipynb` in Jupyter
2. Edit `SYSTEM_PROMPT` (cell 5) and `TASK` (cell 7)
3. Adjust `config.json` as needed
4. Run all cells

## Configuration

`config.json` controls all runtime behaviour:

```json
{
  "model": "qwen3-vl:30b",
  "max_steps": 20,
  "headless": false,
  "use_vision": "auto",
  "flash_mode": true,

  "session_file": null,

  "save_artifacts": true,
  "save_gif": true,
  "gif_frame_ms": 800,

  "human_in_loop": false,
  "confirm_keywords": ["submit", "buy", "delete", "send", "login"],

  "retry_enabled": true,
  "max_retries": 3,
  "retry_delay_s": 2,

  "context_compress_after": 30,

  "schedule": null
}
```

### Key options

| Key | Description |
|-----|-------------|
| `use_vision` | `"false"` hide screenshots · `"true"` auto-capture every step · `"auto"` model requests on demand |
| `flash_mode` | Suppress `<think>` tokens (`think: false` in Ollama API) |
| `session_file` | Path to save/restore browser auth state across runs; `null` = disabled |
| `save_artifacts` | Save `result.json` + `summary.txt` to `./runs/<timestamp>/` |
| `save_gif` | Stitch captured screenshots into `./runs/<timestamp>/replay.gif` |
| `human_in_loop` | Pause before actions matching `confirm_keywords` for manual approval |
| `retry_enabled` | Retry failed tool calls up to `max_retries` times |
| `context_compress_after` | Summarise old message history after N turns; `0` = disabled |
| `model_timeout_s` | Kill the run if Ollama doesn't respond within this many seconds; `0` = disabled |
| `schedule` | Cron expression for recurring runs (e.g. `"0 * * * *"`); `null` = run once |

## Skills

Skills are reusable, parameterised task templates stored in `skills/<name>.yaml`. To use one, set `SKILL` and `SKILL_ARGS` in cell 7 of the notebook (instead of editing `TASK` directly):

```python
SKILL      = "login"
SKILL_ARGS = {"url": "https://example.com/login", "username": "alice", "password": "secret"}
```

Three built-in skills are included:

| Skill | Required args |
|-------|--------------|
| `summarize_page` | `url` |
| `login` | `url`, `username`, `password` |
| `search_and_extract` | `query`, `result_count` |

### Adding your own skill

Create `skills/<name>.yaml`:

```yaml
description: A one-line description of what this skill does.

args:
  url: The target URL

task_template: |
  Go to {url} and do something useful.

system_prompt_extension: |
  Optional extra instructions appended to the base system prompt for this skill only.
```

`task_template` supports standard Python `{variable}` placeholders. Any key listed in `args` must be provided in `SKILL_ARGS` — missing args raise a clear error before the browser or LLM are touched.

Leave `SKILL = None` (the default) to use the raw `TASK` string — existing behaviour is unchanged.

## Orchestrator (multi-agent)

Set `ORCHESTRATOR_TASK` to a high-level goal. An orchestrator LLM (with no browser of its own) decomposes the goal into sub-tasks and delegates each to an isolated sub-agent — every sub-agent gets its own Playwright browser instance.

```python
ORCHESTRATOR_TASK = "Compare the price of a MacBook Pro M4 on Amazon and Best Buy, then recommend the best deal."
SUBAGENT_MODE = "parallel"   # "parallel" | "sequential"
```

The orchestrator calls the `spawn_subagents` tool with a list of tasks and a mode:

| Mode | Behaviour |
|------|-----------|
| `"parallel"` | All sub-tasks start simultaneously — best for independent tasks (e.g. checking multiple sites) |
| `"sequential"` | Sub-tasks run in order; each agent receives prior results as context — best when tasks depend on each other |

The orchestrator LLM decides the mode per batch and can call `spawn_subagents` multiple times, mixing modes. Each sub-agent produces its own `runs/agent<N>_<timestamp>/` folder. The `max_subagents` config key (default `6`) caps how many browsers open per call.

Leave `ORCHESTRATOR_TASK = None` (the default) to use the standard single-agent path.

---

## Credentials

Sensitive values (usernames, passwords, API keys) are stored in gitignored `credentials/<name>.yaml` files:

```yaml
# credentials/github.yaml
username: alice
password: s3cr3t
```

Reference individual values in `TASK` or `SKILL_ARGS` with the `creds:name.key` token syntax:

```python
# With the login skill
SKILL      = "login"
SKILL_ARGS = {
    "url":      "https://github.com/login",
    "username": "creds:github.username",
    "password": "creds:github.password",
}

# With a raw task
TASK = "Go to https://github.com/login and sign in with creds:github.username and creds:github.password"
```

Tokens are expanded before the task text reaches the model — credentials never appear in the notebook or any committed file. See `credentials.example/` for the expected file format.

---

## Features

- **ReAct loop** — Thought → Action → Observation, up to `max_steps` iterations
- **Session persistence** — save/restore cookies and localStorage between runs
- **Run artifacts** — structured JSON result + plain-text summary saved per run
- **GIF replay** — animated replay of every screenshot from the run, displayed inline
- **Human-in-the-loop** — configurable keyword list triggers a confirmation widget before dangerous actions
- **Retry & error recovery** — transient tool failures are retried with backoff
- **Context compression** — older turns are summarised when history grows too long
- **Loop detection** — repeated action fingerprints are caught, re-execution is skipped, and the model is nudged to try a different approach; only unique actions count against the step budget
- **Model timeout** — if Ollama stops responding the run is killed after `model_timeout_s` seconds with a clear error message
- **Task scheduling** — standard cron expressions run the task repeatedly; each run is fully isolated

## Run output

Each run produces a folder at `./runs/<timestamp>/`:

```
runs/
└── 20260315_142301/
    ├── result.json   # structured LLM output
    ├── summary.txt   # plain-text final answer
    └── replay.gif    # animated screenshot replay
```

## Dependencies

```
mcp>=1.26.0
httpx
python-dotenv
nest_asyncio
Pillow
APScheduler
ipywidgets
```
