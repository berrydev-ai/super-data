# Repository Guidelines

## Project Structure & Module Organization
Keep runtime code inside a `super_data/` package at the repository root; group transport adapters under `super_data/transports/`, data access utilities under `super_data/datasets/`, and shared domain logic under `super_data/core/`. Place integration and unit tests in `tests/` mirroring the package layout (`tests/transports/`, `tests/core/`, etc.). Store sample JSON assets and jq tool definitions beneath `datasets/` following the layout in `README.md`, and keep experimental notebooks or scratch scripts out of the tracked tree.

## Build, Test, and Development Commands
- `uv pip install -e .` — install project dependencies in editable mode.
- `uv run python -m super_data --transport stdio` — start the local MCP server entry point.
- `uv run python -m super_data --transport http --port 8080` — run the HTTP transport for integration testing.
- `uv run pytest` — execute the entire test suite with standard reporting.

## Coding Style & Naming Conventions
Use Python 3.12 features, four-space indentation, and PEP 8 as the baseline. Require type hints on public functions and dataclasses or TypedDicts for structured payloads. Name modules with snake_case (`super_data/query_engine.py`) and classes in PascalCase (`QueryEngine`). For jq assets, keep function names lower_snake_case and provide concise `description` fields to aid tooling.

## Testing Guidelines
Favor `pytest` parametrization to cover dataset edge cases and render fixture JSON under `tests/fixtures/`. Name test files `test_<module>.py` and individual tests `test_<behavior>`. Include regression tests whenever adding jq functions or schema migrations, and validate that HTTP and stdio transports share coverage via common mixin helpers. Aim for meaningful assertions over coverage quotas and document non-trivial test data inline.

## Commit & Pull Request Guidelines
Follow the existing short, imperative commit style (`Add HTTP transport scaffolding`). Group related changes into a single commit and include configuration updates alongside the code that depends on them. Pull requests should summarize the change, list manual or automated test results, reference any tracked issues, and attach artifacts such as HTTP session logs or jq outputs when they clarify behavior. Request review from a maintainer familiar with the affected subsystem before merging.

## Security & Configuration Tips
Never commit `.env` files or credentials; rely on `env.example` snippets in documentation instead. Keep default transport settings minimal and require explicit opt-in for remote services. When working with team datasets, scrub personally identifiable information before uploading to shared storage and document any masking strategy in the PR.
