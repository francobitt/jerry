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
| `schedule` | Cron expression for recurring runs (e.g. `"0 * * * *"`); `null` = run once |

## Features

- **ReAct loop** — Thought → Action → Observation, up to `max_steps` iterations
- **Session persistence** — save/restore cookies and localStorage between runs
- **Run artifacts** — structured JSON result + plain-text summary saved per run
- **GIF replay** — animated replay of every screenshot from the run, displayed inline
- **Human-in-the-loop** — configurable keyword list triggers a confirmation widget before dangerous actions
- **Retry & error recovery** — transient tool failures are retried with backoff
- **Context compression** — older turns are summarised when history grows too long
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
