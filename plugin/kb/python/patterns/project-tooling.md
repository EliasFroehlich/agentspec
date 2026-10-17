# Project Tooling (uv, ruff, type checkers)

> **Purpose**: Reproducible Python projects with uv, PEP 735 dependency groups, ruff and a type checker, pinned to the deployment interpreter
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Starting a new package, library or pipeline repository
- Replacing `pip` + `venv` + hand-edited `requirements.txt` with a lockfile
- Adding lint, format and type checks to CI
- Keeping local tooling aligned with a fixed runtime (e.g. Databricks Runtime 17.3 LTS = Python 3.12)

## Implementation

```bash
# uv picks the newest installed Python unless told otherwise -> always pass the target
uv init --package my_pipeline --python 3.12   # src/ layout + build backend
cd my_pipeline
uv python pin 3.12                             # writes .python-version

uv add "pydantic>=2.10,<3"                     # runtime dependency
uv add --group test "pytest>=9" pytest-cov     # [dependency-groups] test
uv add --group lint "ruff>=0.17" "mypy>=2"     # [dependency-groups] lint
uv add --dev ipython                           # --dev == --group dev

uv lock                                        # resolve and write uv.lock
uv sync --locked                               # CI: fail if uv.lock is stale
uv run pytest                                  # run inside the project venv
```

### pyproject.toml

```toml
[project]
name = "my-pipeline"
version = "0.1.0"
requires-python = ">=3.12"            # floor; .python-version picks the dev interpreter
dependencies = ["pydantic>=2.10,<3"]  # 2.10.6 is preinstalled on DBR 17.3 LTS

[dependency-groups]                   # PEP 735 (Final): dev-only, never published as extras
test = ["pytest>=9", "pytest-cov"]
lint = ["ruff>=0.17", "mypy>=2"]
dev = [{ include-group = "test" }, { include-group = "lint" }]

[tool.ruff]
target-version = "py312"              # pyupgrade (UP) rules stop at 3.12 syntax
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "RUF", "ANN", "C4", "PT"]

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest]                         # pytest 9+ native TOML table
minversion = "9.0"
testpaths = ["tests"]
strict = true
markers = ["slow: long-running tests", "integration: needs external services"]
```

`uv init --package` also writes a `[build-system]` table; keep the one it generates.

## Configuration

| Setting | Recommended | Why |
|---------|-------------|-----|
| `requires-python` | lowest interpreter you deploy on | resolver refuses dependencies that drop it |
| `.python-version` | same as the runtime (e.g. `3.12`) | without it uv may pick 3.14/3.15 locally |
| ruff `target-version` | `py312` for DBR 17.3 LTS code | UP rules won't suggest 3.13+ syntax |
| ruff `line-length` | `88` | ruff default; matches `specs/python-standards.yaml` |
| `tool.uv.default-groups` | default is `["dev"]` | which groups `uv sync`/`uv run` install |
| pytest `strict` | `true` only with a pinned pytest | future strictness options turn on automatically |

## Tool Status (2026-10-17)

| Tool | Version | Status / note |
|------|---------|---------------|
| uv | 0.13.x | Replaces pip, pip-tools, pipx, poetry, pyenv, twine, virtualenv |
| ruff | 0.17.x | Linter + formatter (Flake8, isort, pyupgrade, Black-compatible format); ANN101/ANN102 removed in 0.8.0 |
| mypy | 2.4.0 | Mature checker; `strict = true` |
| pyright | -- | Also preinstalled on DBR 17.3 LTS (1.1.394) |
| ty | 0.0.89 | Beta, "does not yet have a stable API"; try it, do not gate CI on it yet |

## Example Usage

```bash
# CI job
uv sync --locked --group lint --group test
uv run ruff check .
uv run ruff format --check .
uv run mypy src
uv run pytest -ra

# Auto-modernize code (pyupgrade + unused imports), then review the diff
uv run ruff check --select UP,F401 --fix .

# One-off tools and scripts without touching the project
uvx ty check src                               # try the beta checker
uv run --with pandas python scratch.py         # temporary extra dependency
uv add --script etl.py "requests<3"            # PEP 723 inline metadata in the script
uv pip compile pyproject.toml -o requirements.txt   # pinned file for platforms that need one
```

## Migration Notes

| From | To |
|------|----|
| `[tool.poetry.group.dev.dependencies]` / extras used as dev deps | `[dependency-groups]` |
| `ignore = ["ANN101", "ANN102"]` | delete (removed rules; ruff warns "ignoring them has no effect") |
| `[tool.pytest.ini_options]` | `[tool.pytest]` (pytest 9+; both tables cannot coexist) |
| Black + isort + flake8 | `ruff format` + `ruff check --select I` |

## See Also

- [Clean Architecture](../patterns/clean-architecture.md)
- [Python Versions](../concepts/python-versions.md)
- [Python Standards Spec](../specs/python-standards.yaml)

## Sources

- https://docs.astral.sh/uv/
- https://docs.astral.sh/uv/concepts/projects/dependencies/
- https://docs.astral.sh/uv/guides/scripts/
- https://github.com/astral-sh/uv/releases
- https://peps.python.org/pep-0735/
- https://docs.astral.sh/ruff/
- https://docs.astral.sh/ruff/rules/missing-type-self/
- https://github.com/astral-sh/ruff/releases
- https://mypy-lang.org/news.html
- https://astral.sh/blog/ty
- https://github.com/astral-sh/ty
- https://docs.pytest.org/en/stable/reference/reference.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
