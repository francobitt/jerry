# Jerry — Local Browser Automation Agent

## Project overview
Jerry is a local ReAct-style browser automation agent. It runs entirely on-device using an Ollama-hosted LLM and a Chromium browser controlled via the Playwright MCP server. All agent logic lives in a single Jupyter notebook (`agent.ipynb`).

## Stack
- **LLM**: Ollama (`qwen3-vl:30b` by default) — served at `OLLAMA_BASE_URL` from `.env`
- **Browser control**: `@playwright/mcp@latest` npm package, launched as a subprocess via `npx`
- **MCP client**: Python `mcp` SDK (`StdioServerParameters` + `stdio_client`)
- **Notebook runtime**: Jupyter with `nest_asyncio` for async compatibility

## File structure
```
Jerry/
├── agent.ipynb      # All agent logic — edit SYSTEM_PROMPT and TASK here
├── config.json      # Runtime configuration (see below)
├── .env             # OLLAMA_BASE_URL (not committed)
├── .env.example     # Template for .env
├── requirements.txt # Python dependencies
├── CHANGELOG.md     # Version history
└── CLAUDE.md        # This file
```

## Configuration (`config.json`)
| Key | Type | Description |
|-----|------|-------------|
| `model` | string | Ollama model name |
| `max_steps` | int | Hard cap on ReAct loop iterations |
| `headless` | bool | `true` = no browser window |
| `use_vision` | string | `"false"` / `"true"` / `"auto"` |
| `flash_mode` | bool | `true` = suppress `<think>` tokens (`think: false` in Ollama API) |

### `use_vision` behaviour
- `"false"` — screenshot tools hidden from model entirely
- `"true"` — screenshot tool hidden from model; loop auto-captures after every action round and injects as a user message with `images: [base64]`
- `"auto"` — `browser_take_screenshot` exposed as a tool; model calls it on demand

## Environment setup
```bash
brew install node          # Node.js required for npx / @playwright/mcp
pip install -r requirements.txt
cp .env.example .env       # then set OLLAMA_BASE_URL
ollama pull qwen3-vl:30b
```

## Running the agent
1. Open `agent.ipynb` in Jupyter
2. Edit `SYSTEM_PROMPT` (cell 5) and `TASK` (cell 7)
3. Run all cells — the **Run Agent** cell (last) launches the browser and starts the loop

## Architecture notes
- `npx` is resolved at startup via `shutil.which` with fallback to common install paths; a clear error is raised if Node is not installed
- `env=dict(os.environ)` is passed explicitly to `StdioServerParameters` to prevent PATH stripping in Jupyter subprocess environments
- Ollama tool-role messages do not support the `images` field; image observations are attached as a follow-up `role: "user"` message instead
- `tool_call_id` is included in tool response messages when the model provides an `id` on the call (handles both OpenAI-style and bare Ollama responses)
- The MCP session is opened once per `main()` call and shared across all ReAct steps — do not reinitialise per step
