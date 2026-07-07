# Tooling 和 Tests

改 Python tooling、linting、formatting、typing、tests、fixtures 或 CI checks 时，使用本参考。

## 项目发现

编辑 style 或 tooling 前，检查：

- `pyproject.toml`
- `uv.lock`、`poetry.lock`、`pdm.lock`、`requirements*.txt`
- `.python-version`
- `ruff.toml`、`.ruff.toml`、`mypy.ini`、`pyrightconfig.json`、`pytest.ini`
- 现有 `tests/` 结构和 CI 文件

除非用户要求，不替换现有 toolchain。

## 推荐 Toolchain

新项目默认：

- Package/deps：`uv`，commit lockfile，并在 CI 使用 `uv sync --locked`
- Formatting/linting：`ruff format` 和 `ruff check`
- Type checking：`pyright`、`basedpyright` 或 strict `mypy`；Pydantic-heavy model 使用 mypy 时启用 `pydantic.mypy`
- Tests：`pytest`
- Async tests：按框架使用 `pytest-asyncio` 或 AnyIO
- Coverage：有用时使用 `coverage.py` 或 `pytest-cov`
- 维护型 service 的 secondary checks：`pip-audit` 或 OSV Scanner 查已知漏洞，`deptry` 查 dependency drift，`zizmor` 查 GitHub Actions security。

最小 `pyproject.toml` 风格：

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

按项目调规则；不要倾倒巨大 config。

## Formatting 和 Linting

规则：

- 让 `ruff format` 负责格式化。
- 让 `ruff check --fix` 处理安全机械修复。
- import sorting 交给 Ruff/isort。
- 不要手工重排无关文件。
- 不要加 blanket ignore。必要时用窄 per-line ignore，并写明原因。
- 除非项目已有，不要新增与 Ruff 重叠的新 linter。

## Type Checking

规则：

- 变更代码优先 strict checking。
- 不要用 `Any`、`cast` 或 `# type: ignore` 修 type error，除非边界确实动态。
- 使用 `cast` 时，在附近验证或解释 invariant。
- 避免擦掉 signature 的 untyped decorator。
- 复用的 fixture 和 helper function 要有类型。
- 把 `pyright`/`basedpyright` 或 strict `mypy` 当 CI gate。`pyrefly` 或 `ty` 等更快新 checker 只作为辅助反馈，直到项目证明 parity。
- Pydantic model construction 要被检查：mypy 项目优先 Pydantic plugin；type checker 无法表达 runtime constraint 时，测试 model boundary behavior。

## Testing

测试 behavior 和 contract，不测试 implementation detail。

优先：

- Pure function 和 validator。
- Boundary parsing/serialization。
- Database constraint 和 transaction behavior。
- Authorization 和 tenant isolation。
- Error path、rollback path 和 idempotency。
- Bug regression test。
- route 变化时的 API status code、response schema 和 error contract。
- list endpoint 或 serializer 变化时的 query shape 和 eager loading。

规则：

- 对被触碰行为写 focused test。
- 优先 factory/builder，避免大块复制 fixture。
- test data 小而有意义。
- 测试中避免 sleep；使用 clock/fake/time control。
- unit test 避免 network 和真实第三方服务。
- constraint、transaction、SQL expression 或 Postgres type 重要时，用 integration test 覆盖真实 SQL 行为。

## Pytest 风格

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

规则：

- 小输入矩阵使用 `pytest.mark.parametrize`。
- fixture name 显式；除非无法避免环境 cleanup，否则避免 autouse。
- 一个 test 断言一个 behavior，不一定只写一个 assertion。
- test 名描述 behavior：`test_rejects_expired_token`，不要 `test_token_1`。
- exception message 重要时，用 `pytest.raises(..., match=...)`。
- 非平凡 test 遵循 Arrange, Act, Assert。
- mock 留在 I/O boundary。不要测试 mock behavior 代替 production behavior。

## Database Tests

规则：

- 在数据库中测试 database constraint，不只在 mocked service 里测试。
- 每个 test rollback 或用 fast fixture 重建 schema。
- 避免 test 之间共享 mutable database state。
- Postgres-specific behavior 用真实 Postgres：JSONB index、partial index、array、enum/check semantic、concurrency、locking。
- SQLite 不能替代 Postgres-specific code。
- 绝不对 production 或共享 development database 跑测试。
- CI parity 需要时，优先 testcontainers 或项目现有 isolated database fixture。

## API 和 Async Tests

- FastAPI integration test 使用带 `ASGITransport` 的 `httpx.AsyncClient`。
- 按项目一致使用 `pytest-asyncio` 或 AnyIO。
- 外部 HTTP 用 `httpx.MockTransport`、`respx` 或项目现有 adapter mock。
- expiration、retry 和 scheduling logic 使用 frozen/fake time。
- 避免 `sleep`；等待 observable condition 或注入 clock。

## CI 验证顺序

迭代时使用最窄有用检查：

1. 变更代码的 targeted tests。
2. dependency 或 lockfile 变化时跑 `uv sync --locked`。
3. `ruff format --check` 或项目 formatter check。
4. `ruff check`。
5. Type checker。
6. service repo 的 focused security/dependency checks。
7. shared behavior 变化时跑 full test suite。

报告无法运行的命令及原因。
