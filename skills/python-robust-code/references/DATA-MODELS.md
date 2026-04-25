# Data Models

Choose the right tool for each shape of data. Mixing them is fine; using
`dict[str, Any]` everywhere is not.

## Quick decision

| Shape                                                  | Use                              |
| ------------------------------------------------------ | -------------------------------- |
| Internal value object, fully under your control        | `@dataclass(frozen=True, slots=True)` |
| Input from outside (HTTP, JSON, YAML, env, files)      | `pydantic.BaseModel`             |
| Settings from env vars / `.env`                        | `pydantic_settings.BaseSettings` |
| A finite set of named values                           | `enum.Enum` / `enum.StrEnum`     |
| Closed string variants at API edges (no runtime check) | `Literal["a", "b"]`              |
| Existing dict shape you can't replace                  | `TypedDict`                      |
| Strongly-typed alias for a primitive                   | `NewType("UserId", int)`         |
| Heavy customization or converters                      | `attrs`                          |

## Dataclasses — the default container

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True, kw_only=True)
class Address:
    street: str
    city: str
    postal_code: str
    country: str = "US"
```

Why these flags:

- `frozen=True`: immutability prevents action-at-a-distance bugs and lets
  the value be hashed.
- `slots=True`: ~30% memory savings, plus typo'd attribute writes raise
  instead of silently creating a new attribute.
- `kw_only=True`: forces keyword arguments, which is robust against field
  reordering.

To mutate, use `dataclasses.replace`:

```python
from dataclasses import replace
new_addr = replace(addr, city="Brooklyn")
```

Don't use `frozen` if the dataclass owns a mutable resource (a connection,
a buffer); mutate flag-by-flag in that case.

## Pydantic — for boundary parsing/validation

When data comes from outside, parse it with pydantic v2:

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr, Field, field_validator

class CreateUser(BaseModel):
    model_config = {"frozen": True, "extra": "forbid"}

    email: EmailStr
    display_name: str = Field(min_length=1, max_length=100)
    created_at: datetime
    tags: list[str] = Field(default_factory=list)

    @field_validator("display_name")
    @classmethod
    def strip(cls, v: str) -> str:
        return v.strip()
```

Key choices:

- `extra="forbid"` — unknown fields raise. This catches typos in incoming
  JSON.
- `frozen=True` — immutable instances; same benefits as frozen dataclasses.
- Use `EmailStr`, `HttpUrl`, `IPvAnyAddress`, etc. — they're validated.
- Use `Field(...)` for constraints (`min_length`, `gt`, `pattern`).
- Validators only on real transformations.

Parse at the boundary, then convert to a domain `dataclass` if you don't
want pydantic in your domain layer:

```python
def to_domain(payload: CreateUser) -> User:
    return User(email=payload.email, name=payload.display_name)
```

## pydantic-settings — configuration

Centralize all config in one settings class. Read once at startup; pass it
around explicitly.

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="MYAPP_",
        frozen=True,
        extra="forbid",
    )

    database_url: str
    log_level: str = "INFO"
    request_timeout_s: float = Field(default=10.0, gt=0)
```

- **Never** sprinkle `os.environ.get(...)` across the codebase. One
  settings class is the source of truth.
- Pass `settings` to constructors, don't import a global.
- Mark secrets with `pydantic.SecretStr` so they don't print in tracebacks.

## Enum — closed sets in domain

```python
from enum import Enum, StrEnum, auto

class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"
```

`StrEnum` (3.11+) gives `OrderStatus.PAID == "paid"`, useful for JSON. Use
plain `Enum` if you don't want this implicit equality.

For internal-only enums where the wire format doesn't matter:

```python
class Priority(Enum):
    LOW = auto()
    MEDIUM = auto()
    HIGH = auto()
```

Pattern-match on enums (with `assert_never` for exhaustiveness — see
`python-types`).

## Literal — closed sets at the wire

For something that's a string in JSON and a string in Python:

```python
from typing import Literal

Mode = Literal["read", "write", "append"]
```

Lighter than `Enum`, no class definition. Best for narrow API params.

## TypedDict — typed dicts

When you must keep the data as a dict:

```python
from typing import TypedDict, NotRequired

class UserRow(TypedDict):
    id: int
    email: str
    deleted_at: NotRequired[str | None]
```

`TypedDict` is checked statically only — there's no runtime validation.
For validation, use pydantic.

## NewType — domain primitives without runtime cost

```python
from typing import NewType

UserId = NewType("UserId", int)
OrgId = NewType("OrgId", int)

def get_user(id: UserId) -> User: ...
```

The type checker now refuses `get_user(some_org_id)`. At runtime, `UserId`
is just `int` — zero overhead.

## attrs — when you need more

`attrs` is the spiritual ancestor of `dataclasses`, with extras:

- `field(converter=...)` — automatic type conversion at construction.
- More flexible default factories.
- `validators=[...]` — runtime validation.

Reach for `attrs` when you need converters (e.g., always coerce a `str` to
a `Path` at construction), or you're already invested in `attrs` from a
legacy codebase.

For new projects, `dataclasses` + pydantic at the edge usually covers it.

## Composition over inheritance

Most "I need a base class" instincts are better served by:

- A `Protocol` defining the interface.
- A function that takes the dependency as a parameter.
- A small dataclass holding shared fields, embedded in the larger class.

Inheritance ties lifetimes together. Composition keeps them decoupled and
testable.

## Avoid

- `dict[str, Any]` as a return type — the type-hint version of "I gave up".
- `**kwargs: Any` for public APIs. Make the keyword args explicit.
- Adding fields to a dataclass / BaseModel based on caller — make the
  model own the shape.
- Mutating a "frozen" model by `object.__setattr__`. You'll regret it.
- Mixing pydantic models and dataclasses for the same concept. Pick one
  per layer.
