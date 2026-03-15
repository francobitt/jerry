# Changelog

## jerry0.2 — 2026-03-15

### Added
- **Session persistence** — save/restore browser auth state between runs via `session_file` config key; uses `browser_save_storage_state` / `browser_load_storage_state` MCP tools
- **Run artifacts** — on completion, structured `result.json` and `summary.txt` saved to `./runs/<timestamp>/`; `save_artifacts` config key
- **GIF replay** — all screenshots captured during a run are stitched into `./runs/<timestamp>/replay.gif` and displayed inline; `save_gif` / `gif_frame_ms` config keys; requires `Pillow`
- **Human-in-the-loop checkpoints** — configurable keyword list triggers an ipywidgets confirmation widget before executing dangerous actions (Proceed / Skip / Stop); `human_in_loop` / `confirm_keywords` config keys
- **Retry & error recovery** — failed tool calls are retried with configurable backoff before surfacing as errors; `retry_enabled` / `max_retries` / `retry_delay_s` config keys
- **Context window compression** — when message history exceeds a threshold, older turns are summarised into a single memory message to prevent token overflow; `context_compress_after` config key (0 = disabled)
- **Task scheduling** — standard cron expressions schedule the task to run repeatedly; each run is fully isolated with its own browser session and run folder; `schedule` config key; requires `APScheduler`
- New dependencies: `Pillow`, `APScheduler`, `ipywidgets`

---

## jerry0.1 — 2026-03-15

Initial release.

### Added
- `agent.ipynb` — Jupyter notebook containing the full ReAct agent loop (Thought → Action → Observation)
- `config.json` — runtime configuration: `model`, `max_steps`, `headless`, `use_vision`, `flash_mode`
- `.env.example` — template for `OLLAMA_BASE_URL`
- `requirements.txt` — Python dependencies (`mcp`, `httpx`, `python-dotenv`, `nest_asyncio`)
- Ollama LLM integration via `/api/chat` with `think: false` support for flash mode
- Playwright MCP server integration via `npx @playwright/mcp@latest` subprocess
- Three vision modes: `"false"` (no screenshots), `"true"` (auto-capture each step), `"auto"` (model-requested)
- `npx` auto-detection with fallback search across Homebrew, nvm, and volta install paths
- Inline screenshot display in the notebook via `IPython.display.Image`
- Graceful max-steps handling with a final summary call to the LLM
