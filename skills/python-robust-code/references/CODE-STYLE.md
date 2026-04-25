# Code Style — readability first

Style is what code looks like when no one is looking. ruff handles
formatting; this file is about the parts ruff can't enforce.

## Naming

- Modules: `snake_case`, short, singular (`order.py`, not `orders_module.py`).
- Classes: `PascalCase`. A class name should read like a noun
  (`OrderRepository`, not `HandleOrders`).
- Functions / methods / variables: `snake_case`. Verbs for actions, nouns
  for values.
- Constants: `UPPER_SNAKE_CASE` at module top.
- Private: leading underscore (`_helper`). Don't use double underscore
  unless you actually want name mangling.
- Booleans: `is_`, `has_`, `should_` prefix (`is_active`, `has_payment`).
- Counts: `n_items` or `item_count`, not `num` or `count` alone.
- Don't abbreviate domain terms. `customer`, not `cust`. `request`, not
  `req`.

## Function shape

A function should fit on a screen. If it doesn't, extract.

**Single responsibility.** If you write "and" in the docstring, split it.

**Early returns over nesting.**

```python
# bad
def grade(score: int) -> str:
    if score >= 0:
        if score < 60:
            return "F"
        else:
            if score < 90:
                return "B"
            else:
                return "A"
    else:
        raise ValueError("negative")

# good
def grade(score: int) -> str:
    if score < 0:
        raise ValueError("negative")
    if score < 60:
        return "F"
    if score < 90:
        return "B"
    return "A"
```

**Few parameters.** Past four arguments wants a dataclass or keyword-only
arguments. Use `*` to force keyword-only:

```python
def render(template: str, *, debug: bool = False, max_width: int = 80) -> str: ...
```

**No flag arguments.** A `bool` parameter that picks between two
behaviors usually means two functions.

## Don't repeat yourself, but don't compress prematurely

**Rule of three**: extract on the third occurrence, not the second. Two
similar code paths that might diverge are fine; an abstraction merging
two unrelated futures is hard to undo.

## EAFP, not LBYL

Pythonic code asks for forgiveness, not permission:

```python
# bad
if "user" in payload:
    name = payload["user"]
else:
    name = "anonymous"

# good
name = payload.get("user", "anonymous")

# bad
if hasattr(obj, "close"):
    obj.close()

# good
try:
    obj.close()
except AttributeError:
    pass
```

But always be specific about the exception type — never bare `except`.

## pathlib over os.path (cross-platform)

```python
# bad
import os
path = os.path.join(os.path.dirname(__file__), "data", "users.json")
with open(path, "r") as f: ...

# good
from pathlib import Path
path = Path(__file__).parent / "data" / "users.json"
data = path.read_text(encoding="utf-8")
```

`pathlib.Path`:

- Hides the `\` vs `/` difference between Windows and POSIX.
- Has methods for nearly everything in `os.path`.
- Tests cleanly with `tmp_path`.

**Always specify `encoding="utf-8"`** on `read_text` / `write_text` /
`open` — defaults differ across platforms (UTF-8 on Linux/macOS,
mbcs/cp1252 on older Windows).

## Dataclasses, frozen + slots

For internal value objects (full guide in
[DATA-MODELS.md](DATA-MODELS.md)):

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Money:
    amount: int
    currency: str
```

## Constants

Magic numbers and strings get names:

```python
# bad
if user.failed_logins >= 5:
    lock(user)

# good
MAX_FAILED_LOGINS = 5
if user.failed_logins >= MAX_FAILED_LOGINS:
    lock(user)
```

For groups of related constants, an `Enum` is better than five
module-level names.

## Comprehensions

Comprehensions are great for filter+map. Stop being clever past one level:

```python
# fine
active_emails = [u.email for u in users if u.is_active]

# unreadable — make it a loop or a generator function
result = [process(x) for sub in data for x in sub if validate(x) and x.score > threshold]
```

For complex transforms, use a `def` and call it from the comprehension.
Generators (`(... for ...)`) are preferred when the result is consumed
once.

## Imports

- Group: stdlib, third-party, first-party. Ruff/isort enforces this.
- No `from x import *`. Ever.
- No relative imports across packages (`from ..other_pkg import …`).
  Sibling-only relative imports inside a package are OK.

## Logging, not printing

```python
import logging
logger = logging.getLogger(__name__)

def process(order: Order) -> None:
    logger.info("processing order", extra={"order_id": order.id, "amount": order.total})
```

- Configure logging once, at the application entry point.
- Library code creates a logger with `__name__` and emits — never
  configures.
- Never `print` in library code (ruff's `T20` will flag it).
- Use `extra={...}` (or `structlog`) for structured fields rather than
  f-string interpolation.

## Subprocess (cross-platform)

```python
import subprocess
result = subprocess.run(
    ["git", "rev-parse", "HEAD"],
    check=True,
    capture_output=True,
    text=True,
    timeout=10,
)
sha = result.stdout.strip()
```

- **Always** pass a list, never a string. `shell=True` is a code smell
  and a portability footgun (different shells across OSes).
- **Always** `check=True` unless you're inspecting the return code.
- **Always** `text=True` (or `encoding="utf-8"`) — bytes are rarely what
  you want.
- **Always** set a `timeout` for anything that might hang.
- For programs whose name differs by OS (`python` vs `python3`,
  `where` vs `which`), prefer to invoke the project's tool through
  `uv run`, which abstracts that away.

## Time (cross-platform)

- Use `datetime.now(tz=UTC)` — never naive `datetime.now()`. Naive
  datetimes are a long-running source of bugs, and DST behavior differs
  by OS.
- Compare and store in UTC. Convert to local only at display.
- `time.monotonic()` for measuring elapsed time, `time.time()` only for
  "wall clock" needs.

## Mutable defaults

```python
# wrong — the list is shared across calls
def append(item: str, into: list[str] = []) -> list[str]:
    into.append(item)
    return into

# right
def append(item: str, into: list[str] | None = None) -> list[str]:
    if into is None:
        into = []
    into.append(item)
    return into
```

Ruff's `B006` catches this.

## Comments and docstrings

- Default to **no comments**. Most comments rot.
- Write a comment when the **why** is non-obvious: a workaround for a
  specific bug, a hidden invariant, a constraint from upstream.
- Don't write comments that just describe what the code does.
- Public APIs (functions, classes, modules used by other people) get a
  docstring. One line is often enough.

## File / module organization

- Group by **domain**, not by **kind**. `users/` containing `models.py`,
  `service.py`, `repository.py` is better than top-level `models/`,
  `services/`, `repositories/`.
- Avoid `utils.py` / `helpers.py` / `common.py`. Either the function
  belongs to a domain (move it there) or it's truly cross-cutting (give
  it a real name).
- One class per file is overkill for Python. Group small, related classes.
- Keep `__init__.py` minimal — re-export the public API and nothing else.
