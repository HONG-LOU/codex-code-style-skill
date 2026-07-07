# Codex Code Style Skill

Strict modern Python code-style guidance for Codex, focused on Pydantic v2, FastAPI, SQLAlchemy 2.x, PostgreSQL, typing, tests, and pragmatic backend cleanup.

Repository: <https://github.com/HONG-LOU/codex-code-style-skill>

Chinese reading copy: [docs/zh-CN/code-style-skill.md](docs/zh-CN/code-style-skill.md). This copy is for GitHub reading only and is not an installable Codex skill entrypoint.

## Quick Install

Clone this repository into your Codex skills directory as `code-style`.

Windows PowerShell:

```powershell
$skills = "$env:USERPROFILE\.codex\skills"
New-Item -ItemType Directory -Force -Path $skills | Out-Null
git clone https://github.com/HONG-LOU/codex-code-style-skill.git "$skills\code-style"
```

macOS/Linux:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/HONG-LOU/codex-code-style-skill.git ~/.codex/skills/code-style
```

If you already have it installed, update instead:

```bash
git -C ~/.codex/skills/code-style pull
```

Windows equivalent:

```powershell
git -C "$env:USERPROFILE\.codex\skills\code-style" pull
```

Restart Codex after installing or updating skills if the skill list does not refresh automatically.

## Usage

Mention the skill in a prompt:

```text
$code-style
Review this Python backend change for style, Pydantic v2 usage, typing, tests, and API boundaries.
```

Typical triggers:

- Python backend implementation or refactor
- FastAPI router, schema, service, or settings work
- Pydantic v2 validation and serialization decisions
- SQLAlchemy 2.x and PostgreSQL query/model changes
- pytest, ruff, pyright, mypy, uv, and CI tooling decisions
- cleanup of raw dicts, `Any`, dynamic attributes, stale comments, or oversized files

## What It Enforces

- Pydantic v2 APIs only, including `ConfigDict`, `TypeAdapter`, validators, serializers, and explicit alias behavior.
- Modern typing: Python 3.12+ syntax, explicit return types, no bare collections, narrow `Any`.
- Thin FastAPI routers, typed schemas, service-owned orchestration, global error handling.
- SQLAlchemy 2.x typed ORM and Postgres constraints for integrity.
- Tooling defaults: `uv`, `ruff`, pyright/basedpyright or strict mypy, pytest, async test support.
- Production hygiene: locked installs, security/dependency checks, focused tests before broad validation.

## Repository Layout

```text
SKILL.md                         Main trigger and operating rules
agents/openai.yaml               UI metadata
references/backend-api.md        FastAPI/API guidance
references/pydantic-v2.md        Pydantic v2 guidance
references/sqlalchemy-postgres.md SQLAlchemy and Postgres guidance
references/performance-concurrency.md
references/tooling-tests.md
references/delivery.md
```

## Notes

This is intentionally strict. It works best for maintained Python services where correctness, explicit contracts, and small verifiable changes matter more than fast throwaway scripting.
