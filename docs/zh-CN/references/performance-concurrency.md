# 性能和并发风格

改 database query、list endpoint、relationship loading、caching、background work、locking、idempotency、connection pool、slow path 或 high-concurrency code 时，使用本参考。

## 数据库查询

- 消除 N+1 query。collection 用 `selectinload`；小的 singular relationship 在能减少 round trip 时用 `joinedload`。
- async SQLAlchemy 中避免隐式 lazy loading。序列化前塑造数据。
- list 或 export path 不需要完整 ORM entity 时，只 select 需要的 column。
- 不要为了 count 加载 row；使用 `select(func.count())`。
- public list endpoint 加 hard cap。大型 feed 使用 cursor/keyset pagination。
- 给常见 `WHERE`、`JOIN` 和 `ORDER BY` predicate 加 index。真实 query 匹配时使用 composite、partial、expression 或 GIN index。

## 事务和锁

- 事务保持短且显式。
- race-prone uniqueness 使用 unique constraint 加处理过的 `IntegrityError`。
- `SELECT ... FOR UPDATE` 只用于真实 critical section，例如 inventory、balance、quota 或 queue assignment。
- lost update 重要但 contention 低时，使用 optimistic locking/version column。
- mutation endpoint 和 background job 用 idempotency key、unique constraint 或 processed-event table 做 idempotent。

## Async 和 Background Work

- 使用 `asyncio.TaskGroup` 做 structured concurrency。当 sibling cancellation 或 exception propagation 重要时，避免裸 `asyncio.gather`。
- 不要在 concurrent task 之间共享 SQLAlchemy session。
- request latency 重要时，把 email、export、PDF、image processing、third-party API retry 和 heavy aggregation 放到 background work。
- retryable work 使用 durable queue。FastAPI background task 只适合简单 best-effort 工作。

## Caching

- cache reference data 和 expensive read-heavy result，并设置清晰 TTL。
- mutation 时，如果 stale data 会违反用户预期，就 invalidate 或 update cache。
- typed cache boundary 使用 `model_dump_json()` 存 Pydantic JSON，并用 `model_validate_json()` parse。
- permission-sensitive data 不要在 key 缺少 tenant/user scope 时 cache。

## Observability

- 有用时记录 duration、query count 或 slow query context。
- request 和 job flow 贯穿 request ID 或 trace ID。
- 复杂优化前先 profile。优先简单 query 和 indexing 修复。
