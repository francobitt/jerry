# Changelog

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
