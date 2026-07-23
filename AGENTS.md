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
- Ollama (local LLM at `localhost:11434`) powers the AI text features (website analysis,
  guided interview, ADN generation, campaign generation). It is installed in this
  environment; the app also boots fine without it (those features just error until it's up).
  Running it here has two non-obvious caveats:
  - No systemd in this VM, so start it manually: `ollama serve` (run in a persistent
    tmux session / background). Health check: `curl -s http://127.0.0.1:11434/api/tags`.
  - This Xeon advertises `AMX_INT8` via CPUID but the sandbox does not permit AMX tile
    instructions, so the default `llama-server` runner segfaults ("AMX model buffer" then
    SIGSEGV) on every generation. Workaround already applied: the AMX CPU backend was
    disabled by renaming `/usr/local/lib/ollama/libggml-cpu-sapphirerapids.so` to
    `.disabled`, forcing ggml to fall back to an AVX-512 backend. If Ollama is reinstalled
    (which restores that file), re-apply this rename. `zstd` is required by the installer.
- Models: `llama3.2:3b` (fast on CPU) and `llama3.1:8b` are pulled. On startup, if Ollama
  is reachable the app auto-pulls its default/required model `llama3.1:8b` in the
  background. Generation uses config `default_model`; set it to `llama3.2:3b` via
  `PUT /api/config {"default_model":"llama3.2:3b"}` for much faster responses on CPU
  (8B interview turns take ~60-90s; 3B ~15-35s).
- The embedded image engine (HuggingFace Diffusers, CPU torch) works without Ollama but
  downloads model weights (~2-4 GB) on first image generation.
- `python3.12-venv` is a required system package for creating the venv (already present
  in this environment).
