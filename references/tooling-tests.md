# Tooling And Tests

Use this reference when changing Python tooling, linting, formatting, typing, tests, fixtures, or CI checks.

## Project Discovery

Before editing style or tooling, inspect:

- `pyproject.toml`
- `uv.lock`, `poetry.lock`, `pdm.lock`, `requirements*.txt`
- `.python-version`
- `ruff.toml`, `.ruff.toml`, `mypy.ini`, `pyrightconfig.json`, `pytest.ini`
- existing `tests/` structure and CI files

Do not replace an existing toolchain unless the user asks.

## Preferred Toolchain

When starting fresh:

- Package/deps: `uv` with committed lockfiles and `uv sync --locked` in CI
- Formatting/linting: `ruff format` and `ruff check`
- Type checking: `pyright`, `basedpyright`, or strict `mypy`; enable `pydantic.mypy` when using mypy with Pydantic-heavy models
- Tests: `pytest`
- Async tests: `pytest-asyncio` or AnyIO depending on framework
- Coverage: `coverage.py` or `pytest-cov` when useful
- Secondary checks for maintained services: `pip-audit` or OSV Scanner for known vulnerabilities, `deptry` for dependency drift, and `zizmor` for GitHub Actions security.

Minimal `pyproject.toml` style:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = [
  "A", "ASYNC", "B", "C4", "DTZ", "E", "F", "FA", "FAST", "I", "N",
  "PERF", "PIE", "PT", "PTH", "RET", "RUF", "SIM", "T20", "UP", "W",
]
ignore = []

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

Tune rules to the project; do not dump huge configs.

## Formatting And Linting

Rules:

- Let `ruff format` own formatting.
- Let `ruff check --fix` handle safe mechanical fixes.
- Keep imports sorted by Ruff/isort.
- Do not manually reformat unrelated files.
- Do not add blanket ignores. Use narrow per-line ignores with a reason only when necessary.
- Avoid adding new linters that duplicate Ruff unless the project already uses them.

## Type Checking

Rules:

- Prefer strict checking for changed code.
- Do not fix type errors with `Any`, `cast`, or `# type: ignore` unless the boundary is genuinely dynamic.
- If using `cast`, validate or explain the invariant nearby.
- Avoid untyped decorators that erase signatures.
- Type fixtures and helper functions when they are reused.
- Treat `pyright`/`basedpyright` or strict `mypy` as the CI gate. Use faster emerging checkers such as `pyrefly` or `ty` as auxiliary feedback until the project has proven parity.
- Keep Pydantic model construction checked: prefer the mypy Pydantic plugin for mypy projects, and test model boundary behavior when the type checker cannot express runtime constraints.

## Testing

Write tests around behavior and contracts, not implementation details.

Prioritize:

- Pure functions and validators.
- Boundary parsing/serialization.
- Database constraints and transaction behavior.
- Authorization and tenant isolation.
- Error paths, rollback paths, and idempotency.
- Regression tests for bugs.
- API status codes, response schemas, and error contracts when routes change.
- Query shape and eager loading when list endpoints or serializers change.

Rules:

- Use focused tests for the touched behavior.
- Prefer factories/builders over large copied fixtures.
- Keep test data small and meaningful.
- Avoid sleeping in tests; use clocks/fakes/time controls.
- Avoid network and real third-party services in unit tests.
- Use integration tests for real SQL behavior when constraints, transactions, SQL expressions, or Postgres types matter.

## Pytest Style

```python
import pytest


@pytest.mark.parametrize(
    ("raw_email", "normalized"),
    [
        ("USER@example.com", "user@example.com"),
        (" user@example.com ", "user@example.com"),
    ],
)
def test_normalizes_email(raw_email: str, normalized: str) -> None:
    assert normalize_email(raw_email) == normalized
```

Rules:

- Use `pytest.mark.parametrize` for small input matrices.
- Use explicit fixture names; avoid autouse except for unavoidable environment cleanup.
- Assert one behavior per test, not necessarily one assertion.
- Name tests by behavior: `test_rejects_expired_token`, not `test_token_1`.
- Use `pytest.raises(..., match=...)` for exception messages that matter.
- Follow Arrange, Act, Assert organization for non-trivial tests.
- Keep mocks at I/O boundaries. Do not test mock behavior instead of production behavior.

## Database Tests

Rules:

- Test database constraints in the database, not only in mocked services.
- Roll back each test or recreate schema with fast fixtures.
- Avoid sharing mutable database state across tests.
- Use real Postgres for Postgres-specific behavior: JSONB indexes, partial indexes, arrays, enum/check semantics, concurrency, locking.
- SQLite is not an adequate substitute for Postgres-specific code.
- Never run tests against production or shared development databases.
- Prefer testcontainers or the project's existing isolated database fixtures for CI parity.

## API And Async Tests

- Use `httpx.AsyncClient` with `ASGITransport` for FastAPI integration tests.
- Use `pytest-asyncio` or AnyIO consistently with the project.
- Mock external HTTP with `httpx.MockTransport`, `respx`, or the project's existing adapter.
- Freeze or fake time for expiration, retry, and scheduling logic.
- Avoid `sleep` in tests; wait on observable conditions or inject clocks.

## CI Verification Order

Use the narrowest useful checks during iteration:

1. Targeted tests for changed code.
2. `uv sync --locked` when dependencies or lockfiles changed.
3. `ruff format --check` or project formatter check.
4. `ruff check`.
5. Type checker.
6. Focused security/dependency checks for service repos.
7. Full test suite when shared behavior changed.

Report commands that could not run and why.
