# AGENTS.md

## Cursor Cloud specific instructions

CCS Brand Assistant is a single Python product: a FastAPI backend (`server/app.py`)
serving a vanilla JS SPA (`app/index.html`). Data is stored as JSON files under
`data/` (no SQL database). It is normally packaged as a Pinokio plugin, but for dev
you run it directly. The update script has already installed dependencies into
`venv/` (see `scripts/setup_venv.sh`), so use `./venv/bin/python`.

### Running the app (dev)
- Start: `PORT=7860 ./venv/bin/python server/app.py`
- UI: `http://127.0.0.1:<PORT>/ui/index.html` · Health: `/api/health`
- Port comes from `--port`, then `PORT` env, else default `7860`.

### Tests / lint
- Test suite: `./venv/bin/python -m pytest tests/` (254 tests; they mock Ollama and
  do not need a running server or network).
- There is no configured linter (flake8/ruff/black); `tests/test_mac.sh` is a static
  validation script. Note it hard-codes the API server URL to port `42003` and checks
  for `self.session.url` in `start.json`, so its "servidor"/`start.json` assertions can
  report FAIL against current repo state even when the app is healthy — rely on pytest.

### Non-obvious gotchas
- Tests write brand/campaign JSON into the real repo `data/` dir. The app's main
  `DATA_DIR` is `<repo>/data` and does not honor `CCS_DATA_DIR` (only the audit log path
  does), so after running pytest you may see leftover test brands in the UI. `data/` is
  gitignored — delete it to reset to a clean state (the app recreates dirs + copies
  `defaults/` on startup).
- `POST /api/brands` expects `target_markets` as a **string** (e.g. `"Chile"`), not an
  array — sending an array returns a 422 validation error.
- Ollama (local LLM at `localhost:11434`) is optional and is NOT installed in this
  environment. The app boots fine without it, but AI text features (website analysis,
  guided interview, ADN generation, campaign generation) require Ollama with a
  `llama3.x` model pulled. The embedded image engine (HuggingFace Diffusers, CPU torch)
  works without Ollama but downloads model weights (~2-4 GB) on first image generation.
- `python3.12-venv` is a required system package for creating the venv (already present
  in this environment).
