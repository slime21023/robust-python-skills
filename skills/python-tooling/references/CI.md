# CI templates

The same four checks run identically on Windows, Linux, and macOS runners.
Choose the runner that matches where your code will actually run.

## GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python: ["3.11", "3.12"]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set Python version
        run: uv python install ${{ matrix.python }}

      - name: Sync (frozen)
        run: uv sync --frozen --all-groups

      - name: ruff format check
        run: uv run ruff format --check .

      - name: ruff lint
        run: uv run ruff check .

      - name: ty
        run: uv run ty check

      - name: pytest
        run: uv run pytest --cov-report=xml
```

`uv sync --frozen` fails if `uv.lock` is stale relative to `pyproject.toml`
— this is what you want in CI. Drop `--all-groups` if you only want the
default group's deps installed.

The matrix runs on three OSes; if you don't need cross-platform validation,
drop the matrix and use `ubuntu-latest` only.

## GitLab CI

`.gitlab-ci.yml`:

```yaml
stages: [check]

variables:
  UV_CACHE_DIR: .uv-cache

cache:
  paths:
    - .uv-cache/

check:
  stage: check
  image: python:3.11-slim
  before_script:
    - pip install --no-cache-dir uv
    - uv sync --frozen --all-groups
  script:
    - uv run ruff format --check .
    - uv run ruff check .
    - uv run ty check
    - uv run pytest --cov-report=term-missing
```

## Generic / shell runner

The whole CI pipeline collapses to four commands. On any runner that has
`uv` available:

```bash
uv sync --frozen --all-groups
uv run ruff format --check .
uv run ruff check .
uv run ty check
uv run pytest
```

## Installing uv on a runner

- **Linux / macOS**:
  `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Windows PowerShell**:
  `irm https://astral.sh/uv/install.ps1 | iex`
- **Any platform with pip**:
  `pip install uv` (slower than the official installers but reliable)

## Running ty in CI

`ty` is pre-1.0. If a CI machine cannot install it (e.g. an internal
mirror without it), substitute the matching mypy or pyright command:

```bash
uv run mypy --strict src tests
# or
uv run pyright
```

## Coverage reporting

`pytest-cov` writes XML with `--cov-report=xml`. To upload to Codecov:

```yaml
- uses: codecov/codecov-action@v4
  with:
    files: ./coverage.xml
```

For Cobertura/Sonar, swap the report format. The point is to fail the
build only on `--cov-fail-under` and let humans look at the report.

## Caching

`astral-sh/setup-uv@v3` enables a uv-aware cache automatically. On other CI
systems, cache `.uv-cache/` (Linux/macOS) or `%LOCALAPPDATA%\uv\cache`
(Windows).
