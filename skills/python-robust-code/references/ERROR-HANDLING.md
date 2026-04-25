# Error Handling

Robust code fails predictably. Predictable failure means: the error has a
name, carries enough context to debug, propagates to a layer that can
decide what to do, and never silently corrupts state.

## Define a project exception hierarchy

Every non-trivial project should have an `errors.py` module:

```python
# src/my_project/errors.py
class AppError(Exception):
    """Base class for all application errors."""

class ConfigError(AppError):
    """Invalid or missing configuration."""

class NotFoundError(AppError):
    """A requested entity does not exist."""

class ValidationError(AppError):
    """Input failed validation at a domain boundary."""

class ExternalServiceError(AppError):
    """An upstream service failed in a way we can't recover from here."""
```

Why:

- Callers can catch `AppError` to handle anything from your code, while
  letting truly unexpected exceptions propagate.
- Specific subclasses let middleware/CLI/HTTP layers map errors to the
  right response.
- Names tell the reader what went wrong. `raise ValueError("user 42")` is
  much weaker than `raise NotFoundError("user 42")`.

Don't go overboard — five or six exception classes is enough for most
projects.

## Raise with context, chain with `from`

```python
def load_config(path: Path) -> Config:
    try:
        raw = path.read_text(encoding="utf-8")
    except OSError as exc:
        raise ConfigError(f"could not read {path}") from exc
    try:
        return Config.model_validate_json(raw)
    except ValidationError as exc:
        raise ConfigError(f"config at {path} is invalid") from exc
```

- `raise X from Y` preserves the original cause in the traceback. Always
  do this when re-raising.
- Use `from None` only when the original exception is genuinely
  uninteresting and would mislead the reader.

## Catch narrowly

```python
# bad — silently eats every error including KeyboardInterrupt and bugs
try:
    do_thing()
except Exception:
    pass

# bad — too broad, almost certainly hides a real bug
try:
    user_id = int(payload["user_id"])
except Exception:
    user_id = 0

# good
try:
    user_id = int(payload["user_id"])
except (KeyError, ValueError) as exc:
    raise ValidationError("payload missing or has non-int user_id") from exc
```

Rules:

- Never bare `except:` (catches `KeyboardInterrupt` and `SystemExit`).
- Never `except Exception:` without re-raising or logging at error level.
- The narrower the catch, the easier the bug is to find.

## Don't catch what you can't handle

If a function can't actually do anything useful with an exception, let it
propagate. Catching at the wrong layer turns a stack trace into a silent
corruption.

Catch at the **boundary** where you can make a decision:

- CLI: print a friendly message and exit non-zero.
- HTTP handler: map to a status code.
- Background worker: log + nack and let the queue retry.

## Validation at the edge, not everywhere

Once data has been validated at the boundary (HTTP request, JSON file, env
var), the inside of your application should **trust** it.

```python
# at the edge — validate
order = Order.model_validate(payload)

# inside — trust
def total(order: Order) -> Money:
    return sum(line.amount for line in order.lines)
```

Sprinkling `if x is None: raise` everywhere inside the app means you
didn't make the type system enforce non-null.

## Don't return error sentinels

```python
# bad — caller has to remember to check
def parse(s: str) -> User | None: ...

# bad — returning -1 / "" / {} for failure
def find_index(items: list[str], target: str) -> int: ...   # -1 means not found

# good
def parse(s: str) -> User: ...   # raises ValidationError on bad input
```

`Optional` is fine when "no result" is a normal outcome (`find` /
`lookup`). For things that *should* succeed, raise.

## Resource cleanup with context managers

Always use `with`:

```python
with path.open("rb") as f:
    data = f.read()

with sqlite3.connect(db_path) as conn:
    conn.execute(...)
```

For your own resources, write a context manager:

```python
from contextlib import contextmanager

@contextmanager
def transaction(conn):
    cursor = conn.cursor()
    try:
        yield cursor
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

## ExceptionGroup (3.11+)

When several things can fail concurrently (TaskGroup, batch operations),
Python collects them in an `ExceptionGroup`:

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_a())
        tg.create_task(fetch_b())
except* TimeoutError as eg:
    log.warning("timeouts: %s", eg.exceptions)
except* ExternalServiceError as eg:
    raise   # re-raise the group
```

`except*` matches the type *inside* the group; the rest passes through.

## Retries

Don't hand-roll retry loops. Use `tenacity`:

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=0.5, max=5),
    retry=retry_if_exception_type(ExternalServiceError),
    reraise=True,
)
def call_upstream(...) -> Response: ...
```

- Only retry **idempotent** operations.
- Cap the total time, not just the count.
- Add jitter (`wait_random_exponential`) for distributed systems.
- Always `reraise=True` so the original exception surfaces.

## Logging on the way up

Log **once**, at the layer that decides what to do. Don't log-and-reraise
at every level — that produces five copies of the same traceback.

```python
def handle_request(req):
    try:
        return service.process(req)
    except AppError as exc:
        logger.warning("request failed", exc_info=exc, extra={"req_id": req.id})
        return error_response(exc)
    except Exception:
        logger.exception("unhandled error", extra={"req_id": req.id})
        raise
```

`logger.exception(...)` is `logger.error(..., exc_info=True)` — use it
inside `except` blocks.

## assert is for invariants, not validation

```python
# wrong — asserts are stripped under python -O
def withdraw(account, amount):
    assert amount > 0, "amount must be positive"

# right
def withdraw(account, amount):
    if amount <= 0:
        raise ValidationError("amount must be positive")
```

Use `assert` to document invariants the type system can't express, not to
validate user input.

## Common smells

- `try` block with five lines of unrelated work — narrow it to the line
  that can actually fail.
- `pass` in an `except` block with no comment — at minimum log; usually a
  bug.
- Catching `Exception` then re-raising the same thing — delete the
  try/except.
- `raise Exception("...")` — pick a real type.
- Repeated `if response.status_code != 200: raise ...` — use
  `response.raise_for_status()` (httpx/requests) or wrap once.
