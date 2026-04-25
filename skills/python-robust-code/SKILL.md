---
name: python-robust-code
description: Make Python code robust and readable — error handling (custom exception hierarchies, narrow except, raise…from, retries, ExceptionGroup), data modeling (dataclass vs pydantic vs TypedDict vs Enum), and code style (naming, function shape, EAFP, pathlib, subprocess, logging, mutable defaults). Use when the user asks to "make this Python robust", "review this Python code", "refactor for clarity", "improve error handling", "design these models", "clean up this module", or when reviewing code with bare except, dict[str, Any], shell=True, prints in libraries, or magic numbers. Cross-platform.
license: MIT
compatibility: Python 3.11+. Cross-platform.
metadata:
  bundle: robust-python
  version: "1.0"
---

# python-robust-code

Make Python code that fails predictably and reads cleanly. This skill
covers three intertwined topics: **error handling**, **data modeling**, and
**code style**. Each has a focused reference.

## When to use

Activate when the user asks to:

- Make Python code more robust / cleaner / more readable
- Review Python code (general code review, not specifically tooling)
- Refactor for clarity, single-responsibility, or maintainability
- Improve or design error handling — exceptions, retries, validation
- Choose between dataclass / pydantic / TypedDict / Enum / NewType
- Clean up a messy module, function, or class
- Audit Python for anti-patterns

For type hints in particular, use `python-types`.
For test-related quality (fixtures, parametrize, hypothesis), use
`python-testing`.
For tool configuration (ruff, ty), use `python-tooling`.

## Core principles

These apply to every Python project. Each branch of this skill expands
them:

1. **Boundaries are validated, internals are trusted.** Use Pydantic /
   explicit parsing at the edges (HTTP, files, env vars, subprocess).
   Inside, pass plain dataclasses or typed values — no defensive
   re-validation.
2. **Make illegal states unrepresentable.** Prefer enums, `Literal[...]`,
   `NewType`, frozen dataclasses, and narrow types over `str` /
   `dict[str, Any]` everywhere.
3. **Fail loudly and specifically.** Custom exceptions with context. Never
   `except:` or `except Exception:` without re-raise. Never silently
   `pass`.
4. **Pure where possible, side effects at the edges.** Keep I/O (network,
   disk, time, randomness) out of computation functions so they're
   trivially testable.
5. **Small, named units.** Functions ≤ ~30 lines, one job. Modules grouped
   by domain, not by kind (no `utils.py` dump).
6. **Readable beats clever.** Explicit names, early returns, no nested
   ternaries, no comprehensions doing five things.
7. **Comments justify, code describes.** Default to no comments. Add one
   only when the *why* is non-obvious.

## Decision: which sub-reference to read

Match the user's request to one of the three references:

| User concern                                        | Read                                         |
| --------------------------------------------------- | -------------------------------------------- |
| Exceptions, retries, validation, error semantics    | [references/ERROR-HANDLING.md](references/ERROR-HANDLING.md) |
| Choosing dataclass / pydantic / Enum / TypedDict    | [references/DATA-MODELS.md](references/DATA-MODELS.md)       |
| Naming, function shape, pathlib, subprocess, logging | [references/CODE-STYLE.md](references/CODE-STYLE.md)         |

For a general "review this code" request, scan all three for relevant
issues — most reviews touch all three areas.

## Quick decision guide

| Situation                                        | Default choice                                                    |
| ------------------------------------------------ | ----------------------------------------------------------------- |
| Internal data container                          | `@dataclass(frozen=True, slots=True)`                             |
| Parsing user / HTTP / file / env input           | `pydantic.BaseModel` (v2) or `pydantic-settings` for env          |
| A finite set of values                           | `enum.Enum` or `Literal["a", "b"]`                                |
| Two semantically different `int`s/`str`s         | `NewType("UserId", int)`                                          |
| Optional dependency on a behavior                | `typing.Protocol`                                                 |
| "Sometimes returns nothing"                      | `T \| None` with explicit `is None` check                          |
| "Returns success or failure with detail"         | Return value for success; raise a specific exception on failure   |
| Building a path                                  | `pathlib.Path`, never string concatenation or `os.path.join`      |
| Logging                                          | `logging.getLogger(__name__)`, never `print` in libraries         |
| Subprocess                                       | `subprocess.run(..., check=True, text=True)`, never `shell=True`  |
| Concurrency for I/O                              | `asyncio` (`anyio` if portable interface is needed)               |
| Concurrency for CPU                              | `concurrent.futures.ProcessPoolExecutor`                          |
| Configuration                                    | `pydantic-settings` reading `.env` + environment                  |

## Anti-patterns to flag and remove

When reviewing, actively call these out:

- `from module import *`
- `except:` or bare `except Exception:` without re-raise/log
- Mutable default arguments (`def f(x=[])`)
- `dict[str, Any]` as a public type — use `TypedDict` or a model
- `# type: ignore` without a specific rule code and a comment
- `os.path` mixed with `pathlib`
- `print` in library code
- Magic numbers / strings — promote to a named constant or `Enum`
- Hand-rolled retry/backoff when `tenacity` exists
- Catching an exception just to re-raise the same one
- `sys.path` manipulation
- Top-level side effects at import time
- `shell=True` in `subprocess`
- Naive `datetime.now()` instead of `datetime.now(tz=UTC)`

## Cross-platform notes

This skill emits no shell commands itself, so all guidance applies on
Windows, Linux, and macOS. The few platform-touching topics:

- **Paths**: always `pathlib.Path`. It hides the `\` vs `/` difference.
  Never join paths with string concatenation or hardcoded separators.
- **Subprocess**: pass a list, set `text=True`, set a `timeout`, and avoid
  `shell=True` — those choices are the same on every OS.
- **File encodings**: always pass `encoding="utf-8"` to `open` /
  `read_text` / `write_text`. The default differs across platforms (UTF-8
  on Linux/macOS, mbcs/cp1252 on older Windows).
- **Newlines**: when reading text files cross-platform, prefer
  `path.read_text(encoding="utf-8")` over manual splitting; let Python's
  universal newlines handle CRLF/LF.
- **Datetimes**: use timezone-aware datetimes (`datetime.now(tz=UTC)`) —
  naive datetimes interact badly with daylight saving on every platform.

## Workflow for a "make this robust" request

1. **Read the code first**. Don't apply rules without understanding intent.
2. **Identify the boundary**. What's user/external input vs internal data?
3. **Apply error handling** (see [ERROR-HANDLING.md](references/ERROR-HANDLING.md)):
   custom exceptions, narrow `except`, `raise … from`.
4. **Tighten the data model** (see [DATA-MODELS.md](references/DATA-MODELS.md)):
   replace dicts with dataclasses or pydantic, magic strings with enums.
5. **Improve the shape** (see [CODE-STYLE.md](references/CODE-STYLE.md)):
   early returns, naming, function size.
6. **Run the tooling** (`uv run ruff check . --fix && uv run ty check`)
   to catch what's mechanical.
7. **Verify behavior** with tests before declaring done.
