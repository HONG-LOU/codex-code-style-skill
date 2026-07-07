# Code Style 中文阅读版

> 仅供 GitHub 阅读。真正被 Codex 加载的英文入口仍是仓库根目录的 `SKILL.md`。不要把本文件复制成全局 skill 的根 `SKILL.md` 使用。

每次触碰 Python 代码、项目布局、仓库配置、测试或技术文档时，都应用这套规则。目标是正确、小、契约明确、可靠、并且兼容当前项目。

核心原则：代码和文件都必须有类型、短、直接、可复用、明显。项目结构应让边界一眼可见：HTTP 入口、业务用例、领域规则、持久化、外部适配器、测试、配置和文档都应放在最小且清晰的位置。如果原始数据、动态访问、手工转换或放错层的逻辑进入业务逻辑，就把验证、归一化和编排推回正确边界。

## 操作规则

1. 先读本地项目：`pyproject.toml`、lockfile、lint/type/test 配置、现有包结构、代表性模块、文档和测试。
2. 遵循现有可工作的模式，除非它们和正确性、安全性、现代 API 或用户明确要求冲突。
3. 优先用最小改动完整解决问题。不要引入 framework、base class、service、repository、factory、metaclass、decorator 或 middleware，除非它们现在就能消除真实复杂度。
4. 用类型、约束模型、数据库约束和聚焦函数让非法状态不可表达。
5. 公共边界严格明确，内部保持简单。
6. 除非项目要求，不要兼容过时 Python 或库版本。
7. 在边界把外部原始数据转换为 typed object。不要让原始字典穿过 service、builder、repository 或 domain 逻辑。
8. 检查触碰代码的直接上游和下游：调用方、被调用方、附近测试、共享 schema、兄弟模块、包边界、配置、文档和重复的本地模式。
9. 直接相关、低风险、可验证时，改进相邻代码或文件组织。不要扩展成无关重写。
10. 重构前先定义保持行为不变的边界。行为变更必须明确、被请求或确实必要，并且有测试。
11. 触碰附近代码时，删除无用代码、注释、metadata、空白和多余字符。
12. 先运行最窄但有意义的验证命令；影响面更大时再跑更宽检查。

## 现代基线

项目没有声明版本时，默认偏好：

- Python：目标 3.12+ 语法；只有项目声明时才用 3.13/3.14 特性。
- Pydantic：只用 v2 API。
- SQLAlchemy：只用 2.x typed ORM 和 Core API。
- PostgreSQL：使用原生数据库约束和类型；数据库能保证完整性时，不写只在应用层生效的完整性规则。
- Tooling：`uv`、`ruff`、`pyright` 或 strict `mypy`，有 async 测试时用 `pytest-asyncio`。
- CI hygiene：维护型服务要有 locked install、type checking、tests、dependency/security checks。

不要为了更新语法而搅动项目。只现代化被触碰的代码或任务所需代码。

## 项目组织

改 package layout、router、service、repository、dependency、settings、script、docs、migration、tests、CI 或根目录配置前，阅读 [references/project-organization.md](references/project-organization.md)。

硬默认：

- 架构应能从目录树读出来。非平凡服务优先使用清晰的 feature/domain package；小项目保持简单，直到额外层次能移除真实复杂度。
- router 和其他 entrypoint 要薄。触碰相关行为路径时，把业务决策、编排、事务和外部 workflow 逻辑移到 service 或 use-case function。
- import 方向保持单向：entrypoint/adapter -> service/use case -> domain -> persistence/external adapter。内层不得 import FastAPI、CLI、job 或 router module。
- 持久化代码放在 repository、query helper 或 data module；它们不做业务决策，也不依赖 HTTP 类型。
- repo root 保持无聊且清晰：package code、tests、docs、migrations、scripts、lockfile 和工具配置都应有明显归属。不要让随机可 import 模块、临时脚本、重复配置或过期文档留在触碰路径附近。
- 改进被触碰路径及其直接上下游的既有组织问题。只有行为有覆盖，或改动是机械且容易验证时，才 rename、move 或 split 文件。
- 不要做宽泛架构迁移、层次创建、`src/` 迁移、package 大搬家或文档重写，除非任务需要且测试能覆盖影响面。

## 代码形态

- 函数只做一件有用的事，并返回 typed value。
- 没有持久状态或多态需求时，优先 module-level function，不要用 class。
- 结构化数据优先小 dataclass 或 Pydantic model；边界避免 untyped dict。
- 通过参数做 dependency injection。避免全局可变状态、隐式 singleton、隐藏 I/O 和 import-time work。
- 名字要准确且朴素：函数用动词，数据用名词，布尔值用 `is_`、`has_`、`should_` 或 `allow_`。
- 查询、验证、事务、错误处理或多个业务概念隐藏在一行里时，不要写聪明 one-liner。

## 文件瘦身

- 文件应小到行为允许的程度。删除 dead code、不可达分支、未使用 import/变量、重复 helper、注释掉的代码、空行噪音和装饰性分隔符。
- 注释是代码重量。默认删除。只保留解释非显然 invariant、事务推理、安全约束、外部协议怪癖，或带明确升级触发条件的限制说明。
- 不要写复述名称、类型、控制流、route 目的、query 目的或下一行代码的注释。改进名字或代码形态。
- 对明显 CRUD endpoint 和简单 query param，不加 FastAPI `description`、`summary`、`response_description`、参数描述或长 docstring。只有非显然契约、外部协议、安全行为、废弃说明或生成 API 要求才保留。
- 同一行为路径里看到附近无用注释和 metadata 时删除；这是预期清理，不是可选 polish。
- 保持公共行为。删除注释、未使用代码或冗余 metadata 不得改变客户端依赖的 API contract、明确要求的生成文档或验证行为。

## 表达式极简

- 优先一个清晰表达式，不写两行机械中间变量。只有能表达真实概念时才引入临时变量、helper、class 或注释。
- 声明式 model 和类型约束优先于命令式验证。Pydantic field constraint 胜过重复 `if`；serializer 胜过手写输出字典。
- 现代 Python 语法能减少噪音时使用：pattern matching、小作用域 assignment expression、comprehension、generator expression、`StrEnum`、`Self`、PEP 695 generic、`type` alias。
- one-liner 只有仍明显时才保留。包含分支、错误处理、事务、查询或多个业务概念时拆开。
- 通过小 typed function、Pydantic model、protocol 和聚焦 query helper 复用。不要为了一个调用者创建含糊 `utils` module 或 abstract base class。
- 字段映射、归一化、序列化或 coercion 重复时，先找 Pydantic model、`TypeAdapter`、validator、serializer、computed field 或 typed helper 来删除重复。
- 写自定义代码前，先检查标准库、框架、当前项目 helper 或已安装依赖是否已经干净解决。

## 相邻代码清理

- 把每个触碰区域当作小维护窗口。检查 caller path、callee path、附近测试、兄弟模块和共享 schema 是否有同类味道。
- 同一行为路径内且可用相同或稍宽测试验证时，修正相邻问题：router 逻辑错位、重复转换、过期命名、raw dict 泄漏、边界验证缺失、无用注释、dead branch、unused code、冗余 FastAPI metadata、不一致 Pydantic 用法、过期 docs/config。
- 保持行为稳定。不要把数据库语义、API contract、auth 行为、事务边界、query cardinality 或错误语义作为顺手清理改变。
- 不要做宽泛 rename、架构迁移、repository 重写或跨切面 style churn，除非任务要求或测试能承受影响面。
- 如果相邻清理有价值但风险高，不要放进 patch，作为 follow-up 提及。

## Backend 和 API

改 FastAPI router、request/response schema、dependency injection、service boundary、settings、auth、error handling、middleware、background task 或 API docs 前，阅读 [references/backend-api.md](references/backend-api.md)。

硬默认：

- router 保持薄：只做 HTTP binding、dependency injection、status code 和 response model。
- 业务编排和事务所有权放在 service。repository/data helper 不做业务决策。
- 使用 Pydantic request、response、create、update、filter 和 settings model，不用 raw dict 或 ad hoc param。
- 使用 domain exception 加 global handler。router 不要只为了转 HTTP response 而 catch service exception。
- 项目拥有 API surface 时，显式声明 `response_model`、status code、pagination limit、auth check 和 security header。
- FastAPI route decorator 和 param 保持精简。避免显然的 `description` 和 `summary`。

## 性能和并发

改 list endpoint、database query、relationship loading、background work、caching、locking、idempotency、connection pool、高并发流或慢路径前，阅读 [references/performance-concurrency.md](references/performance-concurrency.md)。

硬默认：

- 避免 N+1 query 和意外 async lazy loading。序列化前用 eager loading 或 column selection 塑造 query。
- 公共 list endpoint 必须有上限。大型或 append-only feed 优先 cursor/keyset pagination。
- 事务保持短。并发可能造成重复或丢写时，使用 row-level lock、optimistic locking、unique constraint 或 idempotency key。
- 慢外部 I/O 和重 CPU 工作在请求延迟或重试重要时放到 background job。

## 类型规则

- 每个新增或修改的 function、method、class attribute、model field、module-level variable 和 return value 都要有类型。公共边界必须显式。
- 使用内置泛型：`list[str]`、`dict[str, int]`、`tuple[...]`、`set[...]`。
- 不要用 bare `dict`、`list`、`tuple`、`set` 或 `frozenset` annotation。
- 使用 `X | None`；新代码避免 `Optional[X]`。
- `Literal`、`Protocol`、`TypedDict`、`Self`、`Final`、`NewType`、`override` 能让契约更清楚时使用。
- 参数类型优先 `collections.abc`：`Mapping`、`Sequence`、`Iterable`、`Callable`。
- 避免 `Any`。如果 opaque third-party blob、性能敏感 adapter 或有意不验证 passthrough data 必须使用，就限制在最小边界并说明验证或信任边界。未知外部值只在边界用 `object`，不要当业务逻辑捷径。
- 项目目标 Python 3.12+ 时，新 generic code 优先 PEP 695：`class Repo[T]:` 和 `type Row = Mapping[str, object]`。
- 狭窄外部 payload 形状可用 `TypedDict`，当完整 Pydantic model 会增加噪音时使用；进入业务逻辑前仍要在边界验证。
- 不要用 stringly typed status；使用 `StrEnum`、`Literal` 或数据库 enum/check。
- 不要为了满足测试削弱类型；修正生产契约或测试 setup。

## 字典和动态访问

- 不要在业务逻辑中把 raw dict 当结构化数据。JSON、XML/SOAP params、Redis hash、form payload 和 HTTP response 要立即 parse 成 Pydantic model 或其他 typed structure。
- 不要用 `dict.get()` 访问业务字段。它会静默隐藏 typo 和缺失数据。验证后用 typed attribute。
- 合理 dictionary 使用范围要窄：HTTP header、query param、简单 lookup table、typed local map、立即验证前的外部 raw payload，或小局部作用域 SQL row mapping。
- 不要用 constant attribute name 调 `getattr`、`setattr` 或 `hasattr`。用直接属性访问。
- 避免遍历 `model_dump().items()` 后用 `setattr` 更新 ORM object 或 DTO。显式赋字段，让 migration、invariant 和 falsy value 在代码里可见。
- 动态 dispatch 优先 `match`/`case`、typed callable map 或显式 strategy object。不要用 `hasattr` 加 `getattr` 发现方法。
- 不要在业务逻辑里为了读 key 调 `model_dump()`。用 model attribute。dump 只用于 serialization、安全日志摘要、外部 API 或 PATCH diff helper。

## 错误处理

- 边界快速失败。输入验证一次，然后信任 typed internal data。
- 抛窄 domain exception 或标准 exception，消息要可行动。
- 不要 catch broad exception，除非是在补上下文并重新抛出、处理真实 fallback 或 cleanup。
- 不要静默 coerce 损坏数据。使用显式转换方法和测试。
- logging 要结构化且有用。不要记录 secret、token、完整 connection string 或 raw PII。

## Pydantic

创建或修改 Pydantic model、validator、serializer、settings、DTO、API schema 或验证密集代码前，阅读 [references/pydantic-v2.md](references/pydantic-v2.md)。

硬默认：

- 使用 `BaseModel`、`ConfigDict`、`Field`、`Annotated`、`field_validator`、`model_validator`、`field_serializer`、`model_serializer`、`computed_field` 和 `TypeAdapter`。
- 使用 `model_validate`、`model_validate_json`、`model_dump` 和 `model_dump_json`；不要用 v1 API，例如 `parse_obj`、`dict`、`json`、`from_orm`、`@validator` 或 `@root_validator`。
- 简单字段规则优先 `Annotated[..., Field(...)]` constraint、标准 constrained type 和 discriminated union，不手写验证分支。
- 边界 model 如需接受 camelCase、snake_case、PascalCase 或 legacy field name，使用 `AliasChoices` 加 `ConfigDict(validate_by_name=True, validate_by_alias=True)`。新代码不要用 `populate_by_name=True`。
- 依赖 alias 输出的 API schema 要显式设置 `serialize_by_alias`，不要依赖 Pydantic 跨大版本默认值变化。
- 有意选择 strictness：config、ID、money、enum-like protocol 和 internal message 用 `strict=True`；只有用户边界 contract 需要时才允许显式 coercion。
- JSON bytes 或 string 优先 `model_validate_json()`，不要 `json.loads()` 后再 `model_validate()`。
- constrained default 来自动态值或必须像用户输入一样检查时，使用 `validate_default=True`。
- 宽松外部输入 model 在上游会发送无关字段时用 `extra="ignore"`；拒绝未知字段属于 contract 时使用更严格处理。
- PATCH 或 partial-update 流需要区分 omitted field 和显式 falsy value 时，用 `model_fields_set`。
- 独立 list、union、primitive 和外部 payload fragment 验证用 `TypeAdapter`。不要只为了调用 validation 创建 wrapper model。
- 使用 `computed_field`、serializer、alias 和 `model_dump` option 生成 API shape。schema 能表达时，不要逐字段组装 response dict。
- validator 用于边界 coercion 和 normalization。验证后，业务逻辑应使用类型已正确的 attribute。
- typed immutable-style update 更清晰时，用 `model_copy(update=...)`，不要 mutate 或在 dict 间来回 round-trip。
- Pydantic schema 与 SQLAlchemy ORM model 分离。
- Pydantic 用在 I/O 边界，不是每个内部 data object 的替代品。

## SQLAlchemy 和 Postgres

改 ORM model、migration、session、transaction、query code 或 Postgres-specific behavior 前，阅读 [references/sqlalchemy-postgres.md](references/sqlalchemy-postgres.md)。

硬默认：

- 使用 SQLAlchemy 2.x 风格：`DeclarativeBase`、`Mapped[...]`、`mapped_column`、typed `relationship`、`select()`、`Session.execute`、`AsyncSession` 和 `async_sessionmaker`。
- 新代码绝不使用 legacy `Query`、implicit autocommit pattern 或 untyped ORM attribute。
- 每个 unit of work 只有一个清楚的 transaction owner。
- 用 Postgres primary key、foreign key、unique constraint、check constraint、not-null constraint、index 和 migration 保证完整性。
- 真实领域语义用原生 Postgres 类型：`UUID`、`JSONB`、`ARRAY`、`DateTime(timezone=True)`、`Text`、native enum/check 和 money 用 `Numeric`。
- 每次 schema change 都使用 Alembic 或项目 migration 工具；review 自动生成 migration，并在项目支持时验证 upgrade/downgrade。
- 不要混用 ORM entity 和 API response schema。
- 将 Pydantic data 应用到 ORM entity 时显式赋字段。PATCH update 用 `model_fields_set` 或显式 sentinel logic，避免合法 falsy value 丢失。

## Tooling 和 Tests

新增或修改项目 tooling、linting、formatting、typing、tests、fixtures 或 CI 前，阅读 [references/tooling-tests.md](references/tooling-tests.md)。

默认：

- 项目使用 Ruff 时，使用 `ruff format` 和 `ruff check --fix`。
- 优先启用能抓动态属性错误和可避免复杂度的 Ruff rule，包括实际可用时的 `B009` 和 `B010`。
- 使用项目已有 type checker。对变更代码偏严格。
- 使用 `pytest`，纯逻辑写 focused unit test，数据库行为写 integration test。
- 相关时测试 behavior、edge case、validation failure、API contract、authorization、idempotency、transaction rollback path 和 query shape。
- 外部 HTTP 用专门 transport 或库 mock，例如 `httpx.MockTransport` 或 `respx`。unit test 不打真实第三方服务。

## 交付

commit、push、tag、release 或 finalize code change 前，阅读 [references/delivery.md](references/delivery.md)。

硬默认：

- commit 前检查 `git status --short --branch`、`git diff` 和 staged diff。
- 只 stage 目标文件。不要包含无关用户改动。
- commit 前运行仓库验证。如果检查改文件或失败，最小修正后重跑相关检查。
- 除非用户明确要求，不用 destructive git command、amend、force push 或 interactive git flow。
- shipping 后报告 branch、commit hash/message、验证命令和 push 结果。

## 决策启发

- 优先删代码，而不是抽象。
- 优先显式参数，而不是隐藏配置。
- 优先 well-named query function，而不是 generic repository layer，除非项目已使用 repository。
- 优先数据库约束，而不是重复 service check。
- 优先 `TypeAdapter`，而不是 standalone validation 的 wrapper model。
- 简单字段规则优先 `Annotated` constraint，而不是 validator。
- 跨字段 invariant 优先 `model_validator(mode="after")`。
- 优先 `computed_field` 或 serializer，而不是 ad hoc response assembly。
- inbound API payload 优先 `AliasChoices`，而不是手工 key normalization。
- ORM 更新优先显式 attribute assignment，而不是循环 dict copy。
- partial update 中 `0`、`False`、`""` 可能合法时，优先 `model_fields_set`，而不是 truthiness check。
- 优先改进被触碰 caller、callee 和 tests，而不是留下附近直接相关腐坏。
- 错位行为优先移动到最近既有层，而不是发明新架构模式。
- collection relationship 优先 `selectinload` 和显式 eager loading，避免意外 lazy load。
- 持久化 timestamp 优先 timezone-aware `datetime`。
- failures 应取消 sibling 或可预测传播时，优先 structured concurrency（`asyncio.TaskGroup`），而不是裸 `asyncio.gather`。
- 应用设置优先 pydantic-settings。不要在业务逻辑里到处 `os.getenv()`。
- 优先 domain exception、typed result 和 global handler，而不是散落 HTTP-specific catch。

## 避免

- Pydantic v1 API。
- 新 Pydantic model 中的 `populate_by_name=True`。
- SQLAlchemy legacy ORM API。
- `Any`、bare collection annotation 和业务逻辑签名中的 `object`。
- 对业务字段使用 `dict.get()`。
- 对 constant attribute name 使用 `getattr`、`setattr` 或 `hasattr`。
- `model_dump()` 后接 dict subscript 或 generic `setattr` update。
- raw dict 穿过 service、builder、repository 或 domain boundary。
- Pydantic field、validator、serializer、computed field 或 `TypeAdapter` 能替代的 verbose glue code。
- schema 能表达时手工组装 response dictionary。
- 无用注释、注释掉的代码、dead code、装饰性分隔符和空行噪音。
- 显然的 FastAPI `summary`、`description`、`response_description` 或参数描述。
- 过早架构层次。
- 放错位置的业务逻辑：router、dependency、repository、migration、script、config module 或 test。
- 内层 import FastAPI router、CLI/job entrypoint 或 web-only schema。
- 根目录 loose module、重复 config、过期 docs 和模糊跨领域 `utils.py`。
- 无测试覆盖或当前行为路径不需要的宽泛顺手重写。
- 宽泛 `except Exception`。
- 业务逻辑中的 `os.getenv()`。
- unit test 中真实网络调用。
- 无界 public list endpoint 和隐式 relationship lazy load。
- 用 runtime monkeypatch 做应用设计。
- mutable default argument。
- 能用 typed structure 时还用 dynamic attribute 和 magic string。
- 大型含糊 helper module，例如 `utils.py`，除非项目已有聚焦惯例。
- 只改测试来掩盖生产 bug。

## 执行检查清单

完成 Python 工作前，确认 diff 具备：

- 没有新增 `Any`，除非在严格验证的 adapter boundary。
- 没有新增 bare collection annotation。
- 没有新增用于业务字段的 `dict.get()`。
- 没有新增 constant-name `getattr`、`setattr` 或 `hasattr`。
- 没有新增业务逻辑中的 `model_dump()` 加 dict subscript access。
- 没有新增 `populate_by_name=True`；需要时用 `validate_by_name=True` 和 `validate_by_alias=True`。
- 依赖 alias output 的 API schema 显式设置 serialization behavior。
- 外部 raw data 在边界被解析为 typed structure。
- ORM update 显式，尤其是 PATCH flow。
- 使用 Pydantic 或 type-system 特性删除可避免 boilerplate，且不隐藏行为。
- 已检查直接 caller、callee、附近 tests 和 sibling pattern 的直接相关清理。
- 相邻 refactor 限制在保持行为、可验证的范围。
- 已检查触碰路径中的项目组织问题：错位逻辑、错误 layer import、过期 docs/config、模糊 helper module 和 root clutter。
- API/router 变更保持 HTTP concern 与业务逻辑分离。
- Query/list 变更考虑 pagination、eager loading、index 和 transaction scope。
- 文件更瘦：没有新增 dead code、注释掉的代码、无用注释、装饰性分隔符或显然 FastAPI description。
- 项目 lint、format、type 和 test 命令在最窄有意义范围通过。
- Delivery 工作检查 diff，只 stage 目标文件，commit 前验证，并 push 到 tracked remote。
