---
name: python-types
description: Add and improve Python type hints - function annotations, generics (TypeVar/PEP 695), Protocol, TypeGuard/TypeIs, NewType, Literal, TypedDict, Self, and avoiding Any. Use when asked to add type hints, fix type errors, or improve typing. 
license: MIT
compatibility: Python 3.11+. Type checker agnostic — examples work with ty, mypy, and pyright.
metadata:
  bundle: robust-python
  version: "1.0"
---

# python-types

Use type hints to catch a class of bugs before runtime, document interfaces,
and let editors help. Used poorly — `Any` everywhere, `# type: ignore`
sprinkled around — they're worse than nothing.

## When to use

Activate when the user asks to:

- Add or improve type hints on existing code
- Fix a specific type error from ty / mypy / pyright
- Replace `Any` or `dict[str, Any]` with something stronger
- Type a generic / Protocol / overloaded function
- Type asynchronous code, decorators, or higher-order functions
- Decide between `Optional`, `T | None`, `TypeVar`, `Protocol`, `TypedDict`,
  `NewType`, `Literal`, `Enum`

For *configuring* the type checker itself, use `python-tooling`.
For data modeling at the value-object level (dataclass / pydantic), use
`python-robust-code`.

## Where to put types

**Always** annotate:

- Public function parameters and return types
- Class attributes (in the body, or via `dataclass`)
- Module-level constants whose type isn't obvious from the literal

**Usually skip**:

- Local variables — the right-hand side already tells you the type
- Trivial `__init__` parameters that just store attributes (use `dataclass`)
- `self` and `cls`

## Modern syntax (Python 3.11+)

Built-in generics, never `typing.List` / `typing.Dict`:

```python
def lookup(ids: list[int]) -> dict[int, User]: ...
def merge(a: tuple[str, ...], b: tuple[str, ...]) -> tuple[str, ...]: ...
```

Union via `|`, never `Union[...]` or `Optional[...]`:

```python
def find(name: str) -> User | None: ...
def parse(value: str | bytes) -> Token: ...
```

For older codebases on 3.9/3.10 using string annotations or
`from __future__ import annotations`, that's fine — be consistent within
a project.

## Don't reach for `Any`

`Any` disables checking. If you find yourself writing it:

1. Can you use a `TypeVar`? (generic over the input type)
2. Can you use a `Protocol`? (structural — "anything with these methods")
3. Can you use a `TypedDict`? (a `dict` with known keys)
4. Can you use `object` and narrow with `isinstance`? ("anything", but the
   checker forces you to narrow before using attributes)

`object` is almost always better than `Any`.

## Generics

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T | None:
    return items[0] if items else None
```

Bounded:

```python
from numbers import Number
NumT = TypeVar("NumT", bound=Number)

def double(x: NumT) -> NumT: ...
```

Python 3.12+ PEP 695 syntax (cleaner):

```python
def first[T](items: list[T]) -> T | None: ...

class Repository[T]:
    def get(self, id: int) -> T | None: ...
```

Pre-3.12 generic class:

```python
from typing import Generic, TypeVar
T = TypeVar("T")
class Repository(Generic[T]):
    def get(self, id: int) -> T | None: ...
```

## Protocol — structural typing

Use `Protocol` instead of abstract base classes when you want "anything
with these methods":

```python
from typing import Protocol

class SupportsRead(Protocol):
    def read(self, size: int = -1, /) -> bytes: ...

def hash_stream(source: SupportsRead) -> str: ...
```

The caller doesn't need to inherit; any compatible object works. This is
how to invert dependencies without inheritance hierarchies.

`@runtime_checkable` enables `isinstance(x, SupportsRead)` but only checks
method *names*, not signatures. Use sparingly.

## TypedDict — typed dicts at boundaries

When a function returns or accepts a JSON-shaped dict:

```python
from typing import TypedDict, NotRequired

class UserPayload(TypedDict):
    id: int
    name: str
    email: NotRequired[str]   # may be absent
```

For domain logic, prefer `dataclass` or `pydantic.BaseModel`. `TypedDict`
shines for literal JSON in/out without runtime cost.

## Literal — finite sets

```python
from typing import Literal

Mode = Literal["read", "write", "append"]

def open_file(path: Path, mode: Mode) -> IO[str]: ...
```

For a closed enum used in business logic, prefer `enum.Enum`. For "this
string can only be one of these three" at API boundaries, `Literal` is
lighter.

## NewType — domain primitives

Stop passing raw `int` and `str` between layers:

```python
from typing import NewType

UserId = NewType("UserId", int)
OrgId = NewType("OrgId", int)

def transfer(source: UserId, target: UserId) -> None: ...
```

Now `transfer(org_id, user_id)` is a type error, not a Tuesday-afternoon
production bug. Zero runtime cost.

## Self — fluent classes

```python
from typing import Self
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Query:
    table: str
    limit: int = 100

    def with_limit(self, n: int) -> Self:
        return replace(self, limit=n)
```

`Self` correctly types subclasses; don't use the class name directly.

## Narrowing

Type checkers narrow types after `isinstance`, `is None`, `assert`, and
explicit checks:

```python
def length(x: str | bytes | None) -> int:
    if x is None:
        return 0
    if isinstance(x, str):
        return len(x.encode())   # x is str here
    return len(x)                # x is bytes here
```

Custom narrowing — `TypeGuard` (3.10+) or the more precise `TypeIs`
(3.13+):

```python
from typing import TypeIs

def is_str_list(values: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(v, str) for v in values)
```

Use `assert_never` for exhaustiveness checks on `match` / `if`/`elif`
chains over enums or `Literal`:

```python
from typing import assert_never

def describe(status: OrderStatus) -> str:
    match status:
        case OrderStatus.PENDING:
            return "pending"
        case OrderStatus.PAID:
            return "paid"
        case _:
            assert_never(status)   # type checker errors if a case is missed
```

## Forward references

If you need to reference a class defined below, either reorder, use
`from __future__ import annotations`, or quote the name:

```python
class Node:
    parent: "Node | None"
```

The `__future__` import makes all annotations lazy strings — good for
forward refs, but breaks tools that introspect annotations at runtime.
Pydantic v2 and dataclasses both work either way.

## Decorators and higher-order functions

Use `ParamSpec` to preserve a callable's signature:

```python
from collections.abc import Callable
from typing import ParamSpec, TypeVar
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")

def timed(fn: Callable[P, R]) -> Callable[P, R]:
    @wraps(fn)
    def inner(*args: P.args, **kwargs: P.kwargs) -> R:
        return fn(*args, **kwargs)
    return inner
```

PEP 695 (3.12+) is even cleaner:

```python
def timed[**P, R](fn: Callable[P, R]) -> Callable[P, R]: ...
```

## Async types

```python
from collections.abc import AsyncIterator, Awaitable

async def fetch(url: str) -> bytes: ...

def stream(urls: list[str]) -> AsyncIterator[bytes]: ...

def maybe_await(x: T | Awaitable[T]) -> Awaitable[T]: ...
```

`AsyncIterator[T]` is the type of `async for ...`; `Awaitable[T]` is the
type of anything you can `await`.

## Suppressing — only with a code and a reason

```python
result = thirdparty_func()  # type: ignore[no-any-return]  # upstream lacks types
```

- Always specify the rule code.
- Always include a comment.
- Bare `# type: ignore` is a code smell — it hides future errors silently.
- `cast(T, value)` is a last resort; prefer narrowing via `isinstance`,
  `TypeGuard`, or `assert`.
- If a third-party library lacks types, write a `.pyi` stub and reuse it
  rather than scattering `cast` calls.

## Avoid these

- `from __future__ import annotations` if your code needs annotations at
  runtime (pydantic with computed fields, custom dataclass field defaults
  that read other fields). Otherwise it's fine.
- `Optional[X]` with a default that isn't `None`. Use `X | None = None`.
- Annotating `self` / `cls`.
- Catching too broad an exception type just to satisfy the type checker.
- Type-aliasing `dict[str, Any]` to make it look intentional. It still
  isn't.
