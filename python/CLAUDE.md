# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a monorepo of **independent ADK (Google Agent Development Kit) sample agents**. Each agent under `agents/<name>/` is a self-contained Python project with its own `pyproject.toml`, `uv.lock`, `.venv`, and `README.md`. There is no shared dependency graph between agents — treat each one as its own project.

The top-level `pyproject.toml` (`/Users/marcus/dev/gcp/adk-samples/pyproject.toml`, one level up from this directory) only defines a shared **ruff/codespell configuration** that individual agents extend via `extend = "../../../pyproject.toml"`. It is not an installable package.

The repo also contains `go/`, `java/`, and `typescript/` sibling trees with their own samples — this CLAUDE.md only covers `python/`.

## Per-agent layout

```
agents/<kebab-case-name>/        # e.g. customer-service
├── <snake_case_name>/           # the importable package, e.g. customer_service/
│   ├── agent.py                 # defines `root_agent` (entry point ADK looks for)
│   ├── prompts.py               # GLOBAL_INSTRUCTION / INSTRUCTION strings
│   ├── config.py                # pydantic-settings Config (env-driven)
│   ├── tools/                   # function tools registered on the agent
│   ├── sub_agents/              # multi-agent setups: each has its own agent.py + prompt.py
│   └── shared_libraries/        # callbacks, helpers
├── deployment/deploy.py         # Vertex AI Agent Engine deploy script
├── eval/                        # eval_data/, sessions/, test_eval.py
├── tests/unit/                  # pytest unit tests
├── pyproject.toml               # uv-managed; extends root ruff config
└── .env.example                 # copy to .env before running
```

The kebab/snake split is required by Poetry/uv packaging — the directory name uses `-` and the Python package uses `_`.

## Common commands (run from the agent's directory, e.g. `agents/customer-service/`)

```bash
uv sync --dev                              # install deps incl. dev tools
uv run adk run <snake_package_name>        # CLI: e.g. uv run adk run customer_service
uv run adk web                             # Dev UI; pick agent from dropdown
uv run pytest tests/unit                   # unit tests
uv run pytest eval                         # eval suite (slower; uses real models)
uv run pytest tests/unit/test_tools.py::test_name   # single test
```

Some older agents use Poetry instead of uv (`poetry install` / `poetry run …`). Check the agent's README; don't assume one or the other.

## Lint/format checks

A shared script at `python-checks.sh` (run from the `python/` directory) enforces black/isort/flake8 on a specific agent path:

```bash
./python-checks.sh --run-all   agents/<agent-name>
./python-checks.sh --run-lint  agents/<agent-name>
./python-checks.sh --run-black agents/<agent-name>
./python-checks.sh --run-isort agents/<agent-name>
```

The path argument **must start with `agents/` or `notebooks/`** — the script rejects anything else. CI runs the same script per changed agent. Many agents also have ruff/mypy configured in their own `pyproject.toml` (`uv run ruff check`, `uv run mypy`).

## Environment variables

Most agents read GCP/Vertex config from `.env` via `pydantic-settings`. Standard keys:

```bash
GOOGLE_CLOUD_PROJECT=...
GOOGLE_CLOUD_LOCATION=us-central1   # some agents now default to us-east1
GOOGLE_GENAI_USE_VERTEXAI=1
```

Agent-specific keys (BigQuery datasets, API keys, dataset IDs, etc.) are documented in each `.env.example` and README — don't invent new env var names.

## Deployment to Vertex AI Agent Engine

Agents that support deployment follow this pattern:

```bash
uv build --wheel --out-dir deployment   # build wheel into deployment/
cd deployment
uv run python deploy.py                 # run from deployment/ so paths resolve
```

`deploy.py` typically reads `GOOGLE_CLOUD_PROJECT` / `GOOGLE_CLOUD_LOCATION` and uploads the wheel.

## ADK conventions to know

- Every agent exposes `root_agent` from its top-level package (`from <pkg> import root_agent`). The ADK CLI / Dev UI discovers agents by importing this symbol.
- Single-agent samples instantiate `google.adk.Agent(model=..., instruction=..., tools=[...])` directly. Multi-agent samples use `SequentialAgent`, `ParallelAgent`, or compose `Agent` instances via `sub_agents=[...]`.
- Tools are plain Python functions with type hints — ADK introspects signatures to build the function-calling schema. Keep docstrings tight; the model sees them.
- Callbacks (`before_agent`, `before_tool`, `after_tool`, `before_model_callback` for rate-limiting) are wired in `shared_libraries/callbacks.py` and registered on the `Agent` constructor.
- `__init__.py` files re-export `agent` / `root_agent` — `**/__init__.py` is exempted from F401 (unused-import) at the root ruff config.

## When working on an individual agent

1. Read that agent's `README.md` first — setup, run commands, and required env vars vary.
2. `cd` into the agent directory; do not try to run things from the repo root.
3. Most agents are mocked (no real backend) — see comments in `tools/tools.py`. Don't add real network calls without checking the README intent.
4. The `agents/README.md` table catalogs every agent with its complexity, vertical, and tags.
