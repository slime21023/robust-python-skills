---
name: python-init
description: Scaffold a new Python project with `uv init --bare` and a small set of copy-in templates (pyproject.toml, .pre-commit-config.yaml, .gitignore). Picks an appropriate layout (single-file script, flat package, or src layout), then assembles the project from templates with no scripts to run. Use when the user asks to "start a Python project", "create a new Python package", "uv init", "bootstrap a Python repo", "set up a new Python service/library/CLI", or "scaffold a Python script". Cross-platform — works on Windows, Linux, and macOS through `uv run`.
license: MIT
compatibility: Requires uv (https://docs.astral.sh/uv/) and Python 3.11+. Cross-platform; no bash, no helper scripts.
metadata:
  bundle: robust-python
  version: "1.1"
---

# python-init

Bootstrap a new Python project from templates. The flow is:

1. `uv init --bare <name>` — create a minimal project skeleton.
2. Copy three templates from this skill's `assets/` over the result.
3. Create the directory tree for the chosen layout.
4. `uv sync` — install dependencies.

There is no helper script. Every step is a single command the agent runs
directly, so the same instructions work on Windows, Linux, and macOS.

## When to use

Activate when the user asks to:

- Start / create / scaffold / bootstrap / init a Python project, package,
  library, CLI, service, or script
- Run `uv init` for a new project
- Set up a Python repo "from scratch"
- Pick a project layout

If the user is **modifying an existing** project (adding ruff/ty/etc. to
something already initialized), use `python-tooling` instead.

## Step 1 — pick a layout

Match the layout to what is actually being built:

| User wants…                                          | Layout          | Bootstrap command                           |
| ---------------------------------------------------- | --------------- | ------------------------------------------- |
| One file run with `uv run script.py`                 | PEP 723 script  | `uv init --script <name>.py`                |
| Internal app / service that won't be installed       | flat package    | `uv init --bare <name>`                     |
| Library, CLI, or service shipped as a wheel          | src layout      | `uv init --bare <name>`                     |
| Repo with multiple cooperating packages              | uv workspace    | `uv init --bare <name>` + workspace config  |

**Default to flat layout** for applications and **src layout** for
anything that will be installed or imported. Migration between flat ↔ src
is a 5-minute change later.

Ask the user if the intent isn't clear from context.

For a **single-file script**, follow Step 2A. For everything else, follow
Step 2B.

## Step 2A — single-file script

```bash
uv init --script <name>.py
uv add --script <name>.py <runtime-deps>
```

`uv init --script` writes a PEP 723 script with inline dependency
declarations. The skill's templates are not needed for this layout —
nothing to copy. Stop here unless the script grows enough to graduate to
a flat layout.

## Step 2B — flat or src layout (template-driven)

### Bootstrap an empty project

```bash
uv init --bare <name>
cd <name>
```

`--bare` creates only `pyproject.toml`, with no source tree, README,
`.python-version`, or sample script. This is the empty canvas the
templates fill in.

### Copy the three templates from this skill

Copy these files from this skill's `assets/` directory into the new
project. Paths are relative to the skill root:

| Source                             | Destination                  | Action                                 |
| ---------------------------------- | ---------------------------- | -------------------------------------- |
| `assets/pyproject.toml`            | `pyproject.toml`             | **Overwrite** the bare one             |
| `assets/pre-commit-config.yaml`    | `.pre-commit-config.yaml`    | Create                                 |
| `assets/gitignore.template`        | `.gitignore`                 | Create                                 |

After copying, **patch `pyproject.toml`** for this project:

1. Replace `name = "my-project"` with the actual project name.
2. Replace every occurrence of `my_project` with the package name (use
   underscores; e.g. `my-project` → `my_project`).
3. For **flat layout**, also:
   - change `packages = ["src/<pkg>"]` → `packages = ["<pkg>"]`
   - change `src = ["src", "tests"]` (under `[tool.ruff]`) → `src = [".", "tests"]`
   - change `include = ["src", "tests"]` (under `[tool.ty.src]`) →
     `include = ["<pkg>", "tests"]`
4. For an **app that won't be published**, you may delete the
   `[build-system]` and `[tool.hatch.*]` sections entirely and run with
   `uv run python -m <pkg>`.

The agent should perform these substitutions with `Edit` directly on the
copied file — no patcher script required.

### Create the directory tree

For **flat layout** (`<pkg>` = the package name with underscores):

```text
<name>/
├── pyproject.toml          (already copied)
├── .pre-commit-config.yaml (already copied)
├── .gitignore              (already copied)
├── <pkg>/
│   └── __init__.py
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

For **src layout**:

```text
<name>/
├── pyproject.toml
├── .pre-commit-config.yaml
├── .gitignore
├── src/
│   └── <pkg>/
│       ├── __init__.py
│       └── py.typed         (libraries only — ships type information)
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

The agent creates these with the appropriate `mkdir` / `touch` /
`Write` calls. Suggested starting `__init__.py`:

```python
"""<pkg> public API."""
__all__: list[str] = []
```

### Sync and install hooks

```bash
uv sync --all-groups
git init                         # if not already a git repo
uv run pre-commit install
```

`pre-commit install` requires a git repository — `uv init --bare` doesn't
create one, so run `git init` first if needed.

## Step 3 — verify the scaffold

Run all four. They should pass on an empty project:

```bash
uv run ruff format --check .
uv run ruff check .
uv run ty check
uv run pytest
```

If `ty` is not yet installable in the user's environment (it is pre-1.0),
swap to `mypy --strict` or `pyright` — see `python-tooling` for the
fallback.

## Common building blocks (add as the project grows)

These are not required at init time; add them when the project actually
calls for them:

- `config.py` — centralized settings via `pydantic-settings`. Add when env
  vars or config files appear.
- `errors.py` — custom exception hierarchy. Add once the call stack has
  more than two layers.
- Domain subpackages (`orders/`, `billing/`, …) — once the package
  exceeds a handful of modules. Group **by domain, not by kind**: avoid
  top-level `models/`, `services/`, `utils/`.
- `py.typed` marker — only for libraries that ship type information.

## Cross-platform notes

- All bootstrap commands go through `uv` and `git` directly — no shell
  features (no pipes, no `&&`-only chains). PowerShell, `cmd`, bash, and
  zsh all run the same lines.
- File copies should use the agent's `Write` / `Edit` tools rather than
  platform-specific `cp` / `Copy-Item` to keep instructions uniform.
- Subsequent `uv run …` commands handle venv activation internally — no
  `Scripts/Activate.ps1` vs `bin/activate` to worry about.
- File paths inside `pyproject.toml` use forward slashes, which work on
  Windows in every modern tool (uv, ruff, ty, pytest, hatchling).

## When to push back

- **Single 50-line script** that won't grow: skip the layout and
  pre-commit; use `uv init --script` and stop.
- **Existing toolchain** (poetry, pip-tools, pyright, black) on the
  user's team: do not swap it out without asking. Suggest a migration
  path; respect their constraints.
- **Notebook-driven work**: notebooks themselves don't fit this template.
  Lift reusable code into a flat or src package and apply this skill
  there only.
