# Python build & test guidelines

Detected via `pyproject.toml`, `requirements.txt`, `setup.py`, or `Pipfile` at the service root.

All commands below run inside the service's Docker container (Step 4.2 of `SKILL.md`), never directly on the host.

## Containerized install & build

- Build the image: `docker build -t <service>-dev .` (or `docker compose build <service>` if the repo has a compose file). Rebuild whenever the Dockerfile or lockfile changes.
- Run every command in this file as: `docker run --rm -v "$(pwd):/app" -w /app <service>-dev <command>`, or `docker compose run --rm <service> <command>` if a compose file defines this service — prefer compose when available since it already carries the service's networking, env vars, and volume mounts.
- Install dependencies inside the container using the package-manager install command below (e.g. `docker run --rm -v "$(pwd):/app" -w /app <service>-dev poetry install`) before running tests, lint, or type-check.

## Package manager

Pick based on what's present — don't introduce a new one:

- `poetry.lock` → `poetry install`, run commands via `poetry run <cmd>`
- `uv.lock` → `uv sync`, run commands via `uv run <cmd>`
- `Pipfile.lock` → `pipenv install --dev`, run commands via `pipenv run <cmd>`
- `requirements.txt` only → `pip install -r requirements.txt` (the container's own Python environment stands in for a host virtualenv — don't create one on the host)

## Test

- Runner: `pytest` (or the repo's configured runner in `pyproject.toml` / `pytest.ini` / `tox.ini`)
- Run only the affected package/module's tests during Step 4 iterations; run the full suite at Step 5.
- Mock external calls with `unittest.mock` / `pytest-mock`, not real network/DB/filesystem access.

## Lint & format

- Format: `black .` (or `ruff format .` if the repo has migrated)
- Lint: `ruff check .` (fallback: `flake8`)
- Apply formatting before considering an issue done; don't hand-format to match a tool's output.

## Type checking

- `mypy .` if `mypy.ini` / `[tool.mypy]` config exists, otherwise `pyright` if configured.
- Treat new type errors introduced by the change as failures, even if the repo has pre-existing unrelated ones.

## Service standards

- Entry point and dependency manifest live at the service directory root (monorepo layout from `CLAUDE.md`).
- Dockerfile: `python:<version>-slim` base matching the pinned interpreter version in `pyproject.toml`/`.python-version`.
- Tracing/metrics: instrument with the OpenTelemetry Python SDK (`opentelemetry-sdk`, `opentelemetry-instrumentation-*` for frameworks in use).
- API spec: keep the OpenAPI spec in sync — if using FastAPI/Flask-smorest, regenerate from the app; otherwise hand-edit the spec file alongside route changes.
</content>
</invoke>
