---
name: python-testing
description: Write and improve Python tests with pytest — layout, naming, AAA structure, fixtures, parametrize, built-in fixtures (tmp_path, monkeypatch, caplog), property-based testing with Hypothesis, async tests, fakes vs mocks, snapshot tests, and what *not* to test. Use when the user asks to "add tests", "write a pytest", "test this function", "fix this test", "add hypothesis tests", "improve test coverage", "test async code", or "set up pytest fixtures". For pytest *configuration* (pyproject.toml, coverage, markers), use `python-tooling`. Cross-platform.
license: MIT
compatibility: pytest 8+, hypothesis 6+. Cross-platform.
metadata:
  bundle: robust-python
  version: "1.0"
---

# python-testing

A test suite is a force for robustness only if it runs fast, fails for one
reason at a time, and is trusted enough that a red CI actually blocks a
merge.

## When to use

Activate when the user asks to:

- Write tests for a function / class / module
- Add or refactor pytest **fixtures**
- Add **parametrize** to existing tests
- Add **Hypothesis** property-based tests
- Test **async** code
- Test code that touches the **filesystem**, **environment variables**,
  **time**, or **subprocess**
- Improve / debug a flaky or slow test
- Decide between fakes, mocks, and patching
- Add **snapshot** tests

For initial pytest *config* (pyproject.toml, coverage threshold, markers),
use `python-tooling`. This skill is about writing the tests themselves.

## Layout

```
tests/
├── conftest.py             # shared fixtures
├── unit/                   # no I/O, no network, fast
│   └── test_<module>.py
└── integration/            # real DB, real HTTP, opt-in
    └── test_<feature>.py
```

- Mirror the source tree under `tests/unit/` so finding the test for
  `src/my_project/orders/service.py` is mechanical.
- Mark integration tests with `@pytest.mark.integration` and skip them by
  default in fast loops (`pytest -m "not integration"`).

## Naming

```python
def test_<unit>_<scenario>_<expected_result>(): ...
```

Examples:

- `test_normalize_strips_whitespace`
- `test_create_user_rejects_duplicate_email`
- `test_retry_gives_up_after_three_attempts`

A reader scanning the file should understand the contract from names alone.

## AAA structure

Arrange, Act, Assert — one of each, in that order, with blank lines:

```python
def test_total_excludes_cancelled_lines():
    order = make_order(lines=[
        line(amount=100, status="active"),
        line(amount=50, status="cancelled"),
    ])

    total = order.total()

    assert total == Money(100, "USD")
```

If a test has multiple Acts, it's two tests. Split it.

## One assertion per concept

Multiple `assert` lines are fine if they're checking the same concept:

```python
result = parse(payload)
assert result.id == 1
assert result.email == "a@b.com"
```

But a test that asserts before *and* after a state change is two tests.

## Fixtures

Put shared setup in `conftest.py`:

```python
# conftest.py
import pytest
from my_project.config import Settings

@pytest.fixture
def settings(tmp_path) -> Settings:
    return Settings(
        database_url=f"sqlite:///{tmp_path}/test.db",
        log_level="DEBUG",
    )

@pytest.fixture
def client(settings):
    from my_project.app import create_app
    return create_app(settings)
```

Scopes:

- `function` (default): fresh per test. Use this unless you have a reason.
- `module` / `session`: for expensive resources. Reset state between tests
  yourself.

Don't accept fixtures you don't use. Don't make a fixture for a one-line
constant.

## Built-in fixtures worth knowing

- `tmp_path` — a fresh `pathlib.Path` directory per test.
  **Cross-platform**: works the same on Windows, Linux, macOS.
- `monkeypatch` — set/unset env vars, attributes, dict keys; auto-undone.
- `caplog` — capture log records and assert on them.
- `capsys` / `capfd` — capture stdout/stderr.

```python
def test_reads_env(monkeypatch):
    monkeypatch.setenv("MYAPP_LOG_LEVEL", "DEBUG")
    settings = Settings()
    assert settings.log_level == "DEBUG"
```

## Parametrize for tables

```python
@pytest.mark.parametrize(
    ("inp", "expected"),
    [
        ("hello", "Hello"),
        ("HELLO", "Hello"),
        ("hElLo", "Hello"),
        ("", ""),
    ],
)
def test_titlecase(inp, expected):
    assert titlecase(inp) == expected
```

Parametrize beats a loop inside a test — failures point to the row.

For complex cases, use `pytest.param` with an `id`:

```python
@pytest.mark.parametrize(
    "payload",
    [
        pytest.param({"user_id": "x"}, id="non-int-user-id"),
        pytest.param({}, id="missing-user-id"),
    ],
)
def test_validation_rejects(payload): ...
```

## Asserting on exceptions

```python
import pytest

def test_withdraw_negative_raises():
    with pytest.raises(ValidationError, match="positive"):
        withdraw(account, amount=-1)
```

- Always provide `match=` — pin the message so a future refactor doesn't
  change the error semantics silently.
- Catch the **specific** exception type. `pytest.raises(Exception)` means
  nothing.

## Fakes > mocks > patches

Order of preference:

1. **Real object** if it's cheap (in-memory SQLite, an actual `Path` under
   `tmp_path`).
2. **Fake** — a hand-written test double implementing the same `Protocol`.
   A fake `EmailSender` that records what was sent is far better than a
   `MagicMock`.
3. **Mock** — `unittest.mock.Mock` for a narrow boundary.
4. **monkeypatch** — replace a function on a module.
5. **patch** (`unittest.mock.patch`) — last resort. Brittle; couples
   tests to import paths.

Mocks easily produce green tests for broken code. The classic failure
mode: production code is renamed; the mock keeps returning the expected
value; the test passes; the integration breaks.

## Property-based testing

For pure functions that should hold an invariant for any input, use
`hypothesis`:

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    once = sorted(xs)
    twice = sorted(once)
    assert once == twice
```

Hypothesis shrinks failing inputs to their minimal form. Worth its weight
for parsers, formatters, encoders, and any "round-trip" property.

## Async tests

```python
import pytest

@pytest.mark.asyncio
async def test_fetch_returns_json(httpx_mock):
    httpx_mock.add_response(json={"ok": True})
    result = await fetch("/api")
    assert result == {"ok": True}
```

- Use `pytest-asyncio`. Setting `asyncio_mode = "auto"` in
  `pyproject.toml` auto-marks every `async def test_*`.
- For HTTP, prefer `respx` (httpx) or `pytest-httpx` over patching the
  client.

## Filesystem tests (cross-platform)

```python
def test_writes_config(tmp_path):
    target = tmp_path / "settings.json"
    write_config(target, {"debug": True})
    assert target.read_text(encoding="utf-8") == '{"debug": true}'
```

- Always use `tmp_path` — never write to a hard-coded path.
- Always pass `encoding="utf-8"` — defaults differ across platforms.
- For directory walking, use `pathlib` methods (`Path.iterdir()`,
  `Path.glob()`), not OS-specific separators.

## Subprocess tests

Mock at the boundary, not at the OS level. If you have a thin `run_git()`
wrapper, fake the wrapper. Don't try to fake `subprocess.run` itself
across operating systems.

## Time-sensitive tests

- Don't `time.sleep` in tests — slow, flaky, doesn't actually exercise
  the case.
- Use `freezegun` or `time-machine` to control wall-clock time.
- For async timers, use `asyncio.run` with a controllable clock or
  `anyio` mock.

## Database tests

- Use a real database engine. SQLite-in-memory for unit tests of
  repositories; the same engine as production for integration tests.
- Wrap each test in a transaction that's rolled back afterward, or
  recreate the schema in a `tmp_path` SQLite per test.
- Don't share state between tests via module-level globals.

## What not to test

- Library code you don't own (don't test that `pydantic` validates
  emails).
- Trivial getters/setters with no logic.
- Print-style output of debug helpers.
- Implementation details (private methods, internal state) — test the
  public contract.

## Coverage as a smell, not a goal

Coverage tells you what's **not** tested; it doesn't tell you what's
tested **well**. Don't write a test just to hit a number — that's how you
get assertion-free tests that exist only to execute lines.

If a function genuinely shouldn't be tested directly, exempt it
explicitly:

```python
if __name__ == "__main__":   # pragma: no cover
    main()
```

## Speed

- Keep unit tests under 10ms each. Suite under 30s.
- Quarantine slow tests with `@pytest.mark.slow` and exclude from default
  runs (`pytest -m "not slow"`).

## Snapshot tests

For HTML/JSON output that's stable but tedious to assert field-by-field:

```python
def test_renders_invoice(snapshot):
    output = render_invoice(invoice)
    assert output == snapshot
```

Use `syrupy`. Snapshot tests are fragile if the output isn't deterministic
— sort dict keys, freeze time, fix random seeds, *then* snapshot.

## When a test fails

A failing test is a signal — investigate before "fixing" it:

- Test was wrong → fix the test, explain in the PR why the old assertion
  was incorrect.
- Code was wrong → fix the code; the test caught a real bug.
- Both — the design is unclear; pause and discuss.

Never "fix" a test by relaxing the assertion just to make CI green.

## Cross-platform notes for tests

- All examples here work on Windows, Linux, and macOS as written.
- `tmp_path` and `pathlib` handle path differences for you.
- `monkeypatch.setenv` works the same on all OSes.
- pytest's command-line invocation goes through `uv run pytest` — same on
  every platform.
- If a test inspects an OS-specific behavior (e.g. file permissions on
  POSIX), guard with `@pytest.mark.skipif(sys.platform == "win32", ...)`.
