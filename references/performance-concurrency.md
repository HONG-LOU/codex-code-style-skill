# Performance And Concurrency Style

Use this reference when changing database queries, list endpoints, relationship loading, caching, background work, locking, idempotency, connection pools, slow paths, or high-concurrency code.

## Database Queries

- Eliminate N+1 queries. Use `selectinload` for collections and `joinedload` for small singular relationships when it reduces round trips.
- In async SQLAlchemy, avoid implicit lazy loading. Shape data before serialization.
- Select only needed columns for list or export paths when full ORM entities are not required.
- Do not load rows just to count; use `select(func.count())`.
- Add hard caps to public list endpoints. Use cursor/keyset pagination for large feeds.
- Add indexes for frequent `WHERE`, `JOIN`, and `ORDER BY` predicates. Use composite, partial, expression, or GIN indexes when they match real queries.

## Transactions And Locks

- Keep transactions short and explicit.
- Use unique constraints plus handled `IntegrityError` for race-prone uniqueness.
- Use `SELECT ... FOR UPDATE` only for real critical sections such as inventory, balances, quotas, or queue assignment.
- Use optimistic locking/version columns where lost updates matter but contention is low.
- Make mutation endpoints and background jobs idempotent with idempotency keys, unique constraints, or processed-event tables.

## Async And Background Work

- Use `asyncio.TaskGroup` for structured concurrency. Avoid bare `asyncio.gather` when sibling cancellation or exception propagation matters.
- Do not share SQLAlchemy sessions across concurrent tasks.
- Offload email, exports, PDFs, image processing, third-party API retries, and heavy aggregations to background work when request latency matters.
- Use durable queues for retryable work. FastAPI background tasks are only for simple best-effort work.

## Caching

- Cache reference data and expensive read-heavy results with clear TTLs.
- Invalidate or update cache on mutations when stale data would violate user expectations.
- Store Pydantic JSON with `model_dump_json()` and parse with `model_validate_json()` for typed cache boundaries.
- Do not cache permission-sensitive data without tenant/user scoping in the key.

## Observability

- Log duration, query count or slow query context where useful.
- Add request IDs or trace IDs through request and job flows.
- Profile before complex optimization. Prefer simple query and indexing fixes first.
