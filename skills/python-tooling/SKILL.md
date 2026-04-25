---
name: python-tooling
description: Configure and run the Python toolchain — ruff (lint+format), ty (type check, with mypy/pyright fallback), pytest, pre-commit, and CI. Use when the user asks to "add ruff", "set up ty", "configure type checking", "add pre-commit", "set up Python CI", "run linting", "fix lint errors", "configure pytest", or "wire up Python tooling" on an existing project. For brand-new projects use `python-init` instead. Cross-platform — works on Windows, Linux, and macOS.
license: MIT
compatibility: Requires uv and Python 3.11+. Cross-platform.
metadata:
  bundle: robust-python
  version: "1.0"
---

# python-tooling

Configure ruff, ty, pytest, pre-commit, and CI on a Python project. The
feedback loop should be fast enough to run on every save.

## When to use

Activate when the user asks to:

- Add or configure **ruff** (lint or format)
- Add or configure **ty**, **mypy**, or **pyright** (type check)
- Add or configure **pytest** (test runner / coverage)
- Add **pre-commit** hooks
- Wire Python tooling into **CI** (GitHub Actions, GitLab CI, etc.)
- Diagnose or fix lint / type / test failures
- Tighten or relax tool configuration

For scaffolding a brand-new project from scratch, use `python-init`.

## The four commands

The whole feedback loop reduces to four commands. Each must work on Windows,
Linux, and macOS — they all do, because they go through `uv run`:

```bash
uv run ruff format .                # format
uv run ruff check . --fix           # lint, auto-fix what's safe
uv run ty check                     # type check
uv run pytest                       # run tests
```

In CI, replace `--fix` with `--check` and `format .` with `format --check .`
so the build fails on drift instead of mutating files.

## Step-by-step

### 1. ruff — lint + format

`ruff` replaces black + isort + flake8 + pyupgrade and most plugin
ecosystems with one Rust binary.

Add to the project:

```bash
uv add --dev ruff
```

Configure in `pyproject.toml`. Use the strict-but-pragmatic rule set in
[references/RUFF.md](references/RUFF.md) — copy that block into the
project's `pyproject.toml` and adjust `target-version` to match
`requires-python`.

When the user has lint errors:

1. Run `uv run ruff check . --fix` first — many fixes are mechanical.
2. For the remaining errors, read each rule code at
   <https://docs.astral.sh/ruff/rules/>. Don't suppress unless the rule is
   genuinely wrong for the situation.
3. To suppress a single occurrence, write `# noqa: <CODE> - reason`. Bare
   `# noqa` is itself a lint error (`RUF100`).

Don't mix ruff with another formatter. Pick one.

### 2. ty — type check (with fallback)

`ty` is Astral's type checker. It is **pre-1.0** as of late 2025; some
configuration keys may shift. If `ty` is unavailable or rejects a config
key, fall back to mypy or pyright (pick **one**).

Add `ty`:

```bash
uv add --dev ty
uv run ty check
```

Configure under `[tool.ty]` in `pyproject.toml`. The recommended baseline:

```toml
[tool.ty.environment]
python-version = "3.11"
python = "./.venv"

[tool.ty.src]
include = ["src", "tests"]
```

Fallback to **mypy**:

```bash
uv add --dev mypy
uv run mypy --strict src tests
```

`pyproject.toml`:
```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_unused_ignores = true
warn_redundant_casts = true
files = ["src", "tests"]
```

Fallback to **pyright**:

```bash
uv add --dev pyright
uv run pyright
```

`pyproject.toml`:
```toml
[tool.pyright]
include = ["src", "tests"]
typeCheckingMode = "strict"
pythonVersion = "3.11"
```

Suppress a single line: `x = foo()  # type: ignore[arg-type]  # reason`.
Always include the rule code and a reason.

For typing patterns themselves (Protocol, generics, narrowing), invoke
`python-types`.

### 3. pytest

```bash
uv add --dev pytest pytest-cov hypothesis
```

Recommended `[tool.pytest.ini_options]` block:

```toml
[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
addopts = [
    "-ra",
    "--strict-config",
    "--strict-markers",
    "--cov=<your_pkg>",
    "--cov-report=term-missing",
    "--cov-fail-under=80",
]
xfail_strict = true
filterwarnings = ["error"]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests requiring external services",
]
```

`--strict-markers` and `--strict-config` turn typos into errors instead of
silent skips. `filterwarnings = ["error"]` makes deprecation warnings fail
tests so they get fixed early.

For pytest patterns themselves (fixtures, parametrize, hypothesis), invoke
`python-testing`.

### 4. pre-commit

```bash
uv add --dev pre-commit
uv run pre-commit install
uv run pre-commit run --all-files       # first-time check
```

Use the cross-platform config in
[references/PRE-COMMIT.md](references/PRE-COMMIT.md) — it normalizes line
endings to LF, runs ruff (lint + format), runs ty via a local hook, and
optionally runs a fast pytest subset on push.

The local-hook approach (`language: system`, `entry: uv run ty check`)
works on Windows, Linux, and macOS without any per-OS configuration.

### 5. CI

See [references/CI.md](references/CI.md) for templates covering GitHub
Actions, GitLab CI, and a generic shell-runner setup. The core pattern is
the same on every CI provider:

1. Install `uv` (one curl/PowerShell command).
2. `uv sync --frozen` — fail if `uv.lock` is stale.
3. Run the four commands in `--check` mode, failing fast on any.

`--frozen` is critical — it ensures CI matches the committed lockfile
exactly.

## Cross-platform considerations

- All commands go through `uv run` — never call the Python binary directly,
  never source `activate`. This sidesteps the `Scripts/Activate.ps1` vs
  `bin/activate` and `python` vs `python3` differences.
- Pre-commit `language: system` hooks run the same `uv run …` command on
  every OS. Avoid `language: docker_image` unless you need it.
- Line endings: rely on `mixed-line-ending --fix=lf` and let git handle the
  rest via `.gitattributes` if needed.
- Path separators in `pyproject.toml` use forward slashes — universal.
- If running on Windows directly (no PowerShell), `cmd` works fine — `uv
  run` does the venv activation internally.

## Order to fix when starting from a messy project

1. **Format first**: `uv run ruff format .`. Removes diff noise from later
   steps.
2. **Auto-fix lint**: `uv run ruff check . --fix`.
3. **Triage remaining lint** by rule group; fix or suppress with reasons.
4. **Add type hints incrementally**: start with `uv run ty check`, fix
   `unresolved-import` and `possibly-unbound-*` first; see `python-types`
   for annotation patterns.
5. **Tests before tightening rules**: get the suite green before turning on
   `filterwarnings = ["error"]` or raising `--cov-fail-under`.
6. **Wire pre-commit and CI** last, when local runs are clean.

## Anti-patterns to flag

- Mixing two formatters (`black` + `ruff format`).
- Mixing two type checkers (`mypy` + `pyright`) on the same project.
- `# noqa` or `# type: ignore` without a code or comment.
- `pip install` in a uv-managed project.
- `pre-commit autoupdate` committed without running tests.
- CI using `pip install -r requirements.txt` when `uv.lock` is committed.
- `pytest -p no:randomly` or other "make CI green" hacks committed without
  understanding the underlying flakiness.
