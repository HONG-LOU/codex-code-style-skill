# Backend 和 FastAPI API 风格

改 FastAPI router、API schema、dependency injection、service、repository、settings、error handling、middleware、OpenAPI docs、auth 或 background work 时，使用本参考。

## 架构

- 先按项目惯例组织。FastAPI service 优先 feature/domain package，边界要清楚。
- router 保持薄：request parsing、dependency、status code 和 response model。
- 业务规则、编排、authorization decision 和 transaction ownership 放在 service。
- repository 或 query helper 聚焦 data access。它们不决定业务结果。
- 通过参数和 FastAPI `Depends` 做 dependency injection；除非项目模式要求，不要在 router 或其他 service 深处实例化 service。
- 内层不得 import 外层：service 不应依赖 FastAPI `Request`、`Response`、`HTTPException` 或 router module。

## API 设计

- collection 使用复数名词，path 使用稳定小写。改约定前遵循现有 path style。
- 读取用 `GET`，创建/action 用 `POST`，partial update 用 `PATCH`，删除用 `DELETE`。
- 除非项目有意 streaming 或返回 raw response，每个 route 都应声明 response type 或 `response_model`。
- create、delete、accepted background work 和 error 显式返回 status code。
- public list endpoint 需要有界 pagination。默认值小，并强制 hard max。
- 不要盲目把所有 response 包成 `{code, data}`，除非现有 API contract 要求。
- 项目支持时使用 RFC 7807-style error body：`detail`、`type`、`instance`。
- decorator 和 param 保持精简。name 和 type 已能解释 contract 时，避免 `summary`、`description`、`response_description` 和参数描述。
- OpenAPI 文本只保留给非显然行为：外部协议怪癖、安全约束、废弃、side effect、特殊 filtering 语义或客户端可见兼容性要求。

## Pydantic Schema

- create、update/patch、filter 和 response 的字段或验证不同时，使用独立 schema。
- 从 ORM object 构造 response schema 时，使用 `ConfigDict(from_attributes=True)`。
- strict client input 使用 `extra="forbid"`，除非 backward/forward compatibility 需要 `ignore`。
- PATCH 语义使用 `model_dump(exclude_unset=True)` 或 `model_fields_set`。
- API route 绝不直接返回 ORM model。

## 错误和安全

- domain exception 定义在靠近 domain 的位置。通过 global handler 映射到 HTTP。
- router 不应只为了翻译异常而 catch service exception。
- mutation 或 expensive work 前检查 authorization。
- 不记录 secret、token、完整 connection string 或 raw PII。
- 生产环境 CORS 要收紧。带 credentials 时不要使用 wildcard origin。
- app 拥有 middleware 时，添加或保留 security header：`X-Content-Type-Options`、`X-Frame-Options`、HTTPS 后的 HSTS、request ID。

## Settings 和 Background Work

- 使用 `pydantic-settings`；启动时验证 config。不要在业务逻辑读 env var。
- FastAPI background task 只用于简单 best-effort 工作。
- retryable、long-running 或 business-critical background job 使用 durable queue。
- background job 保持 idempotent 且 observable。
