# SQLAlchemy 2.x And PostgreSQL Style

Use this reference when creating or changing SQLAlchemy ORM models, Core queries, sessions, transactions, migrations, or PostgreSQL-specific code.

## Core Rules

- Use SQLAlchemy 2.x APIs only.
- Use typed declarative ORM: `DeclarativeBase`, `Mapped[...]`, `mapped_column`, and typed `relationship`.
- Keep ORM models persistence-focused. Keep Pydantic schemas and API contracts separate.
- Make the database enforce integrity. Application checks improve UX; constraints ensure truth.
- Use explicit transaction scopes and one transaction owner per unit of work.
- Prefer simple query functions over generic repositories unless the project already has repositories.

## Declarative Models

```python
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(320), unique=True, index=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
    )

    posts: Mapped[list["Post"]] = relationship(back_populates="author")


class Post(Base):
    __tablename__ = "post"

    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("user_account.id", ondelete="CASCADE"))

    author: Mapped[User] = relationship(back_populates="posts")
```

Rules:

- Use singular or established table naming consistently; do not rename existing tables casually.
- Use `back_populates` rather than implicit backrefs in new code.
- Use `DateTime(timezone=True)` for persisted timestamps.
- Use server defaults for database-owned timestamps and IDs.
- Add `nullable=False` by type where possible; be explicit when clarity matters.
- Avoid default mutable values and Python-side defaults that should be database-owned.

## PostgreSQL Types And Constraints

Use native Postgres types when they express the data better:

- `UUID` for external identifiers.
- `JSONB` for flexible indexed documents, not arbitrary dumping grounds.
- `ARRAY` only when relational modeling would be worse.
- `TIMESTAMP WITH TIME ZONE` through `DateTime(timezone=True)`.
- `TEXT` for unbounded text; `VARCHAR(n)` when the limit is a real contract.
- `NUMERIC` for money or exact decimals; never binary floats for money.
- Native PostgreSQL enums or check constraints for closed sets that the database must enforce.

Use constraints:

```python
from sqlalchemy import CheckConstraint, UniqueConstraint


__table_args__ = (
    UniqueConstraint("tenant_id", "slug", name="uq_project_tenant_slug"),
    CheckConstraint("price_cents >= 0", name="ck_product_price_cents_non_negative"),
)
```

Rules:

- Name constraints and important indexes.
- Add indexes for foreign keys used in joins and predicates, not every column.
- Use partial and expression indexes when they match real query predicates.
- For case-insensitive unique email, prefer a normalized column or a functional unique index.
- Use migrations for every schema change.

## Sessions And Transactions

Async default:

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(settings.database_url, pool_pre_ping=True)
SessionFactory = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionFactory() as session:
        yield session
```

Unit of work:

```python
async def create_user(session: AsyncSession, data: UserCreate) -> User:
    user = User(email=str(data.email), name=data.name)
    session.add(user)
    await session.flush()
    return user
```

Rules:

- The caller owns commit/rollback unless the project has a clear unit-of-work abstraction.
- Use `async with session.begin()` at service/job boundaries.
- Use `flush()` to obtain database-generated values before commit.
- Do not commit inside low-level helpers unless the function is explicitly a full transaction boundary.
- Do not share sessions across concurrent tasks.
- Set `expire_on_commit=False` for async ORM to avoid surprise expired attribute access.

## Query Style

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


async def get_user_by_email(session: AsyncSession, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return await session.scalar(stmt)
```

Rules:

- Use `select()`, `insert()`, `update()`, `delete()`; avoid legacy `Query`.
- Use `session.scalar(stmt)` for one scalar ORM entity or value.
- Use `session.scalars(stmt)` for multiple ORM entities or scalar values.
- Use `session.execute(stmt)` for row tuples/mappings or DML where needed.
- Use `one()`, `one_or_none()`, `first()`, and `all()` intentionally.
- Do not load rows just to count; use `select(func.count())`.
- Avoid N+1 queries; choose loader options explicitly.
- Use bulk `insert().values([...])` or set-based `update()` for bulk work instead of per-row loops when business rules allow it.

## Relationship Loading

- Prefer `selectinload` for collection relationships.
- Prefer `joinedload` for small many-to-one relationships when it reduces round trips.
- Avoid accidental lazy loading during API serialization.
- In async code, be especially explicit because lazy loads can fail outside the expected greenlet/session context.

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.posts)).where(User.id == user_id)
user = await session.scalar(stmt)
```

## Migrations

Rules:

- Use Alembic or the project's existing migration tool.
- Migrations must be reversible unless the project has a documented forward-only policy.
- Include schema constraints, indexes, and server defaults in migrations.
- For large tables, avoid long blocking operations; use concurrent indexes or staged migrations when needed.
- Do not rely on `metadata.create_all()` for production schema changes.
- Review autogenerated migrations before applying them. Autogenerate is a draft, not a source of truth.
- Keep data migrations explicit and idempotent where possible.

## Pydantic Boundary

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr


async def read_user(session: AsyncSession, user_id: int) -> UserRead | None:
    user = await session.get(User, user_id)
    return None if user is None else UserRead.model_validate(user)
```

Rules:

- Convert ORM objects to Pydantic at the boundary.
- Avoid passing ORM objects across layers that do not need persistence behavior.
- Avoid Pydantic validators that query the database.
- Shape queries to include what serialization needs before calling `model_validate`.

## Concurrency And Locking

- Use unique constraints plus retry/handled `IntegrityError` for race-prone uniqueness.
- Use `SELECT ... FOR UPDATE` only when a real critical section is required.
- Keep transactions short.
- Make idempotency explicit for jobs and webhooks.
- Use optimistic locking/version columns when lost updates matter.
- Do not share one `AsyncSession` across concurrent tasks.
- For outbox/event publishing, persist the database change and event atomically before external delivery.

## Avoid

- `session.query(...)` in new code.
- Implicit autocommit assumptions.
- Engine/session creation inside request handlers or hot functions.
- Import-time database connections.
- Broad repository abstractions that only wrap one query each.
- Cascades that delete more than the caller expects.
- JSONB where normal columns would be clearer and safer.
