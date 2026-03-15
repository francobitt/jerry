# Changelog

## jerry0.4.1 — 2026-03-15

### Added
- **Model timeout** — if Ollama does not return a response within `model_timeout_s` seconds the run is killed immediately with a clear error message; set `model_timeout_s` to `0` to disable; default `120`
- New config key: `model_timeout_s` (int)
- `ollama_chat` now uses `MODEL_TIMEOUT_S` as the `httpx` client timeout (replaces the hard-coded 300 s); `asyncio.wait_for` provides a second enforcement layer inside `run_agent` via `_timed_chat`

---

## jerry0.4 — 2026-03-15

### Added
- **Loop detection** — `run_agent` now tracks an action fingerprint (`frozenset` of `(tool_name, sorted_args_json)`) for each step and compares it against the last `loop_window` steps; on a repeat, tool execution is skipped and a nudge is injected so the model tries a different approach; after `max_loop_repeats` consecutive repeats the run ends gracefully with a summary
- Only unique (non-repeated) actions count against the step budget — `steps_taken` in artifacts now reflects distinct actions, not total loop iterations
- Step header now shows both total and unique step counts: `Step N / MAX  (unique: U)`
- Three new config keys: `loop_detection` (bool), `loop_window` (int), `max_loop_repeats` (int)

---

## jerry0.3 — 2026-03-15

### Added
- **Skills system** — reusable, parameterised task templates defined in `skills/<name>.yaml`; each skill specifies a `task_template` with `{variable}` placeholders, optional `system_prompt_extension`, and optional `args` documentation for validation messages
- Three built-in skills: `summarize_page`, `login`, `search_and_extract`
- New notebook cell 8: `load_skill()` + `_resolve_task()` — loads, validates, and renders a skill into a resolved task string and effective system prompt
- `SKILL` / `SKILL_ARGS` module-level variables in cell 7 to invoke skills (leave `SKILL = None` to use the raw `TASK` string — fully backward-compatible)
- `run_agent` now accepts an optional `system_prompt` parameter to support per-skill prompt extensions without mutating the global
- `save_artifacts` now accepts an optional `task` parameter so `result.json` records the rendered task (with real argument values) rather than the raw template string
- Skills work with scheduled runs — `_scheduled_run` also uses `_resolve_task()`
- New dependency: `pyyaml`

---

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
