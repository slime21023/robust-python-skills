# pre-commit — cross-platform configuration

The config below works identically on Windows, Linux, and macOS. Drop into
`.pre-commit-config.yaml` at the repo root.

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-added-large-files
        args: ["--maxkb=500"]
      - id: mixed-line-ending
        args: ["--fix=lf"]   # normalize to LF; works on Windows too

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  # ty does not yet ship a pre-commit hook. Run it via uv from a local hook
  # so it uses the project's pinned version. Swap to mypy/pyright if needed.
  - repo: local
    hooks:
      - id: ty
        name: ty (type check)
        entry: uv run ty check
        language: system
        types: [python]
        pass_filenames: false

      - id: pytest-fast
        name: pytest (fast tests only)
        entry: uv run pytest -m "not slow and not integration" -q
        language: system
        types: [python]
        pass_filenames: false
        stages: [pre-push]
```

## Why `language: system`

`language: system` runs the entry on the host shell. Because we go through
`uv run`, the same string works in PowerShell, `cmd`, bash, and zsh. There
is no need for OS-specific hook configurations.

`pass_filenames: false` means the hook runs on the project as a whole
rather than receiving the staged file list — appropriate for ty and pytest,
which need to see the whole import graph.

## Installing

```bash
uv add --dev pre-commit
uv run pre-commit install                # installs the git hook
uv run pre-commit run --all-files        # one-time check on the existing tree
```

`pre-commit install --hook-type pre-push` also installs the pre-push stage
for the slower `pytest-fast` hook.

## Updating versions

```bash
uv run pre-commit autoupdate
uv run pre-commit run --all-files
```

Always run the full check after `autoupdate` and commit both the config
change and any resulting fixes together.

## Skipping a hook (rarely)

If a single commit needs to skip a hook (e.g. WIP commit):

```bash
SKIP=ty git commit -m "wip"
```

Don't make a habit of `git commit --no-verify` — it bypasses the entire
suite and is what causes pre-commit to feel valuable in the first place.

## Mypy / pyright instead of ty

If using mypy:

```yaml
- repo: local
  hooks:
    - id: mypy
      name: mypy
      entry: uv run mypy --strict src tests
      language: system
      types: [python]
      pass_filenames: false
```

If using pyright:

```yaml
- repo: local
  hooks:
    - id: pyright
      name: pyright
      entry: uv run pyright
      language: system
      types: [python]
      pass_filenames: false
```
