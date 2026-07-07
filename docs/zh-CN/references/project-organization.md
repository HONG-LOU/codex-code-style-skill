# 项目组织

> 仅供 GitHub 阅读。英文运行时参考文件仍在 `references/project-organization.md`。

改 package layout、router、service、repository、dependency wiring、settings、script、docs、migration、tests、CI 或根目录 project config 时，使用本参考。

## 发现和范围

移动文件或改层次前，检查：

- 仓库根文件：`pyproject.toml`、lockfile、tool config、CI、README、docs、scripts、migrations 和 ignore files。
- import root 和 package layout：`src/`、top-level app package、namespace package、app factory 和 entrypoint。
- 现有层次命名：`routers`、`api`、`services`、`domain`、`models`、`schemas`、`repositories`、`queries`、`clients`、`jobs`、`tasks`、`dependencies`、`settings`、`tests`。
- 被触碰行为路径：caller、callee、tests、schemas、database models、migrations、docs，以及共享同一 service logic 的 background/CLI entrypoint。

重构前定义保持行为不变的边界。如果移动会改变 public import、API behavior、database semantics、auth、transaction ownership 或 generated docs，把它当作行为变更并显式测试。

## 仓库根目录

- 根目录保持无聊且可扫读。通常只包含 project metadata、lockfile、docs entrypoint、CI/config file、`src/` 或 app package、`tests/`、`docs/`、`migrations/` 或 `alembic/`，以及聚焦的 `scripts/` 或 `tools/`。
- 项目已使用或新项目启动时，优先把 Python tool config 放进 `pyproject.toml`。只有 tool 或仓库惯例确实受益时，才保留独立 config 文件。
- application 和 service 提交 lockfile。不要手改生成的 lockfile。
- generated artifact、virtualenv、本地 cache、coverage output 和 build output 不进仓库，除非项目有意 vendor。
- 不要把一次性可 import module 留在 repo root。可复用代码进 package，运维脚本进 `scripts/` 或 `tools/`。
- 新维护 package 或需要 packaging/import isolation 的 service 优先 `src/` layout。不要只为风格迁移既有 flat layout，除非 packaging 或 test import 正是问题所在。

## 包布局

选择最小但能明确 ownership 的结构：

- 小应用：紧凑 package 加 `main.py`、`settings.py`、`models.py`、`schemas.py` 和聚焦模块即可。
- 中型服务：优先 feature/domain package；只在确有内容时放本地 `router.py`、`schemas.py`、`service.py`、`repository.py` 或 `queries.py`、`models.py`、`dependencies.py`、`exceptions.py`。
- 共享平台代码：使用明确 shared package，例如 `auth`、`database`、`config`、`observability` 或 `integrations`。不要创建模糊跨领域 `utils.py` 倾倒场。

只有当 layer folder 能降低耦合时才使用。不要创建空的 `services/`、`repositories/` 或 `domain/` 目录做架构表演。

## 层职责

### Entrypoint

Router、CLI command、background job handler、event consumer 和 webhook handler 都是 adapter。它们应该：

- 绑定 protocol 细节：path/body/query param、dependency、auth context、status code、response model、CLI arg、message metadata。
- 在边界验证并转换输入。
- 调一个 service 或 use-case function 完成业务操作。
- 返回 typed response schema 或 exit result。

它们不应该：

- 执行业务 workflow、做持久化决策或拥有多步事务。
- schema 能表达输出时，手工构造 raw response dictionary。
- 只为本地翻译异常而 catch domain/service exception；应由 global handler 或 caller-level mapping 负责。

### Service 和 Use Case

Service 拥有 application behavior：

- 编排 domain rule、repository、external client、authorization decision、idempotency 和 transaction boundary。
- 接受 typed request/use-case data 和 dependencies 作为参数。
- 返回 typed domain result 或 response DTO，不返回 FastAPI `Response`、`Request`、`HTTPException` 或 router-only schema。
- 保持 transaction ownership 显式。一个 use case 协调多个 repository 或 write 时，使用 Unit of Work。

无 durable state 时优先 module-level service function。只有携带有意义 dependency 或符合项目模式时才用 service class。

### Domain

Domain module 包含不依赖 HTTP、database session、environment variable、clock 或 network I/O 就能测试的规则。使用 dataclass、Pydantic model、value object、enum 和 pure function，让非法状态更难表达。

### Persistence

Repository、query helper 和 data module 拥有数据库访问：

- 显式接受 session、connection 或 Unit of Work。
- 按项目惯例返回 ORM entity、row DTO 或 typed result object。
- 保持 query shape、eager loading、locking 和 persistence mapping 可见。
- 不做业务结果决策，不读 HTTP request state，不 import router，不抛 web exception。

只有当 repository 隐藏真实 persistence 复杂度或给 service 更清晰契约时才使用。简单 read path 用 well-named query function 就够。

### 外部适配器

HTTP client、SDK wrapper、queue、file storage、email、payment 和 AI client 应放在聚焦 adapter module 后面。第三方 payload 在 service logic 看到前，要 parse 成 typed boundary model。

## Import 方向

依赖向内流动：

```text
router / cli / job / event adapter
  -> service / use case
  -> domain
  -> repository / query / external adapter interfaces
```

Composition root 负责接线具体 dependency：app factory、dependency provider、worker startup、CLI main 或 test fixture。

除非现有项目有明确例外，否则禁止：

- Service import FastAPI router module、`Request`、`Response`、`Depends` 或 `HTTPException`。
- Domain code import ORM session、web framework、environment config 或 network client。
- Repository import service、router、API schema 或 auth dependency。
- Shared module 只为了少传参数而 import feature module。

## 配置、脚本和文档

- runtime settings 放在一个 typed settings module，优先 `pydantic-settings`。环境变量在启动时进入 settings；业务逻辑接收 typed config 或参数。
- settings 变化时，同步 `.env.example` 或 env table。绝不提交真实 secret。
- operational script 放在 `scripts/` 或 `tools/`。可复用脚本要有 typed `main()`，且没有 import-time side effect。
- migration 放在项目 migration 目录，命名有描述性。review 生成的 migration，避免混入无关 schema change。
- README 作为项目入口：项目是什么、如何 install、configure、run、test 和本地 deploy。
- durable docs 放在 `docs/`。文档增多时，区分 tutorial、how-to guide、reference 和 explanation。不要让文档成为代码或测试已定义行为的第二事实来源。
- 只有当高影响架构决策稳定且未来维护者需要 rationale 时，才写短 ADR-style note。

## 测试

测试镜像行为边界：

- Router/API test 验证 HTTP contract、auth、status code、serialization 和 error mapping。
- Service test 验证 use-case behavior、domain decision、transaction orchestration 和 external adapter boundary。
- Repository/database test 验证 constraint、query shape、lock、migration 和 Postgres-specific behavior。
- CLI/job/event test 验证 adapter parsing 和 delegation to service，不重复测试业务逻辑。

把逻辑从 router 或 script 移出时，把测试移动或新增到现在拥有该行为的层。如果 route/job/CLI contract 重要，在 entrypoint 留一个小 integration test。

## 既有代码清理

在被触碰代码里看到组织味道时：

1. 识别当前行为 owner。
2. 检查直接 caller、callee、tests、schemas 和 docs。
3. 把最小 coherent block 移到最近既有层。
4. 更新同一行为路径内的 import 和 tests。
5. 先跑 focused tests；如果 import 或 shared module 变化，再跑更宽检查。

需要跨 domain move、public import change、database behavior change、大规模 package reshuffle 或缺少测试时，推迟清理并作为 follow-up 提及。

## 味道检查清单

当问题位于被触碰路径且可验证时修复：

- Router 执行多步业务决策、直接 SQL、transaction ownership 或 external workflow orchestration。
- Dependency function 修改业务状态或执行昂贵 workflow logic。
- Service import FastAPI、只为 HTTP 存在的 response schema、CLI parser type 或 job framework object。
- Repository import service、router、auth dependency、API schema，或把 raw dictionary 返回给业务逻辑。
- Domain logic 直接读 environment variable、clock、network、database session 或 global mutable state。
- 同一 use case 在 API、CLI、job 和 event handler 中重复。
- `utils.py` 或 `helpers.py` 混合无关领域。
- 根目录有 loose app module、一次性 script、重复 config file、过期 generated file 或未文档化 required env var。
- 测试只覆盖 entrypoint，而 service/domain 行为已经复杂。

## 来源基础

这些规则参考：

- FastAPI bigger applications 和 APIRouter organization: https://fastapi.tiangolo.com/tutorial/bigger-applications/
- Python Packaging `src` vs flat layout guidance: https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/
- Architecture Patterns with Python service layer、repository 和 Unit of Work pattern: https://www.cosmicpython.com/book/chapter_04_service_layer.html
- Twelve-Factor config guidance: https://12factor.net/config
- pytest good integration practices 和 test layout guidance: https://docs.pytest.org/en/stable/explanation/goodpractices.html
- Ruff configuration guidance: https://docs.astral.sh/ruff/configuration/
- uv project configuration guidance: https://docs.astral.sh/uv/concepts/projects/config/
- Diataxis documentation framework: https://diataxis.fr/
