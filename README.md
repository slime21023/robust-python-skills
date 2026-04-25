# Robust Python — Agent Skills bundle

A set of [Agent Skills](https://agentskills.io) for building robust, readable
Python projects with `uv`, `ruff`, and `ty` (Astral's type checker, with
`mypy` / `pyright` as fallback).

The bundle is split into five focused skills so users can invoke just the part
they need:

| Skill                | When the user wants to…                                              |
| -------------------- | -------------------------------------------------------------------- |
| `python-init`        | Scaffold a new Python project (uv, layout, pyproject template)       |
| `python-tooling`     | Configure or run ruff / ty / pytest / pre-commit / CI                |
| `python-types`       | Add or fix type hints, design typed APIs                             |
| `python-robust-code` | Refactor for robustness — error handling, data models, code style    |
| `python-testing`     | Write or improve pytest tests, fixtures, property-based tests        |

Each skill is a self-contained directory matching the
[Agent Skills specification](https://agentskills.io/specification): a
`SKILL.md` with YAML frontmatter, plus optional `references/` and
`assets/` subdirectories.

## Cross-platform

All skills support **Windows, Linux, and macOS**. There are no scripts to
run; `python-init` is template-driven (`uv init --bare` + copy three files
from `assets/`), and every other skill is documentation that the agent
applies via its own tools.

Where shell commands appear in instructions, they are written to work in any
POSIX shell **and** in Windows PowerShell / `cmd` — typically by going
through `uv run`, which handles activation and path resolution itself.

## Installation

Drop the directory of any skill you want into your agent's skills location.
For example, in Claude Code:

```text
~/.claude/skills/python-init/
~/.claude/skills/python-tooling/
~/.claude/skills/python-types/
~/.claude/skills/python-robust-code/
~/.claude/skills/python-testing/
```

(or `%USERPROFILE%\.claude\skills\…` on Windows)

Each skill is independent — install only the ones you want.

## Validate

```bash
# Install once: pipx install agentskills-cli (or per upstream tool)
skills-ref validate ./python-init
skills-ref validate ./python-tooling
skills-ref validate ./python-types
skills-ref validate ./python-robust-code
skills-ref validate ./python-testing
```

## How the skills work together

These skills are designed to compose. A typical end-to-end flow:

1. `python-init` — bootstrap the project with the right layout.
2. `python-tooling` — confirm ruff / ty / pytest / pre-commit are wired and run.
3. While writing code, the agent draws on `python-types` and
   `python-robust-code` for review and refactoring guidance.
4. `python-testing` — add pytest coverage as features land.

But each skill stands alone — invoke any one without the others.

## License

MIT. See each `SKILL.md` for per-skill metadata.
