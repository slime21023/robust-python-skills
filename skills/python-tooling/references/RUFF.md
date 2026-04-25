# ruff — full configuration reference

Drop the block below into `pyproject.toml`. Adjust `target-version` to
match `[project] requires-python`.

```toml
[tool.ruff]
line-length = 100
target-version = "py311"
src = ["src", "tests"]
extend-exclude = ["build", "dist", ".venv"]

[tool.ruff.format]
docstring-code-format = true
docstring-code-line-length = 80

[tool.ruff.lint]
select = [
    "E", "W",     # pycodestyle
    "F",          # pyflakes (unused imports, undefined names)
    "I",          # isort
    "B",          # bugbear (likely-bug patterns)
    "UP",         # pyupgrade (modernize syntax)
    "SIM",        # simplify
    "RUF",        # ruff-specific
    "N",          # pep8-naming
    "C4",         # comprehensions
    "PT",         # pytest style
    "TCH",        # move type-only imports under TYPE_CHECKING
    "RET",        # return statements
    "PLC", "PLE", "PLW", # pylint conventions / errors / warnings
    "S",          # bandit security
    "ASYNC",      # async pitfalls
    "PIE",        # misc cleanups
    "ERA",        # eradicate dead code (commented-out)
    "PTH",        # prefer pathlib over os.path
    "T20",        # forbid print/pprint in library code
]
ignore = [
    "E501",      # line length is enforced by the formatter
    "S101",      # asserts are fine in tests; per-file override handles src
    "PLW0603",   # `global` is fine when used deliberately
    "RET504",    # explicit assign-then-return is often clearer
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = [
    "S",         # asserts/hardcoded data are expected in tests
    "PLR2004",   # magic numbers in tests are usually fine
    "T20",       # print is OK for debug in tests
]
"scripts/**/*.py" = ["T20"]

[tool.ruff.lint.isort]
known-first-party = ["my_project"]
force-sort-within-sections = true

[tool.ruff.lint.flake8-tidy-imports]
ban-relative-imports = "parents"

[tool.ruff.lint.pydocstyle]
convention = "google"
```

## Adjusting the rule set

The `select` list above is opinionated. To make it lighter:

- Drop `S` if you don't want bandit (security) checks.
- Drop `ERA` if commented-out code is your debugging style during dev.
- Drop `T20` if `print()` is genuinely OK in your codebase (CLIs, scripts).
- Drop `PTH` if you have a large legacy `os.path`-based codebase you can't
  migrate yet.

To make it heavier, consider adding:

- `D` — pydocstyle. Forces docstrings; tune `convention`.
- `ANN` — flake8-annotations. Forces type hints on every function. Heavy.
- `ARG` — flake8-unused-arguments.
- `FBT` — flake8-boolean-trap. Forbids bool positional args.
- `TRY` — tryceratops. Catches common exception-handling smells.
- `LOG` — logging-format issues.
- `DTZ` — datetime-timezone safety. Recommended for any code touching dates.
- `INP` — implicit-namespace-package. Forces `__init__.py`.

## Suppressing rules

Per-line:
```python
result = json.loads(payload)  # noqa: S301 - payload is from trusted internal RPC
```

Per-file (in `pyproject.toml`):
```toml
[tool.ruff.lint.per-file-ignores]
"src/legacy/**" = ["B", "UP"]
```

Per-rule globally:
```toml
[tool.ruff.lint]
ignore = ["RET504"]
```

Always include a comment explaining the suppression. `RUF100` flags
unnecessary `# noqa`, so dead suppressions get cleaned up automatically.

## Format quirks

- Ruff's formatter is black-compatible for most code but not bug-for-bug
  identical.
- `docstring-code-format = true` reformats Python in fenced docstring code
  blocks.
- The formatter and the linter share `pyproject.toml` — they're the same tool.
