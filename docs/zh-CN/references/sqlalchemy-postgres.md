# SQLAlchemy 2.x 和 PostgreSQL 风格

创建或修改 SQLAlchemy ORM model、Core query、session、transaction、migration 或 PostgreSQL-specific code 时，使用本参考。

## 核心规则

- 只使用 SQLAlchemy 2.x API。
- 使用 typed declarative ORM：`DeclarativeBase`、`Mapped[...]`、`mapped_column` 和 typed `relationship`。
- ORM model 聚焦 persistence。Pydantic schema 和 API contract 保持分离。
- 让数据库 enforce integrity。应用检查改善 UX；constraint 保证真相。
- 使用显式 transaction scope，每个 unit of work 一个 transaction owner。
- 除非项目已有 repository，优先简单 query function，而不是 generic repository。

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

规则：

- singular 或既有 table naming 要一致；不要随意 rename existing table。
- 新代码使用 `back_populates`，不要 implicit backref。
- persisted timestamp 使用 `DateTime(timezone=True)`。
- 数据库拥有的 timestamp 和 ID 使用 server default。
- 可能时通过类型添加 `nullable=False`；清晰性重要时显式写出。
- 避免 mutable default 和本应 database-owned 的 Python-side default。

## PostgreSQL 类型和约束

当原生 Postgres type 更能表达数据时使用：

- external identifier 用 `UUID`。
- flexible indexed document 用 `JSONB`，不要当任意倾倒场。
- 只有 relational modeling 更差时才用 `ARRAY`。
- 通过 `DateTime(timezone=True)` 使用 `TIMESTAMP WITH TIME ZONE`。
- unbounded text 用 `TEXT`；limit 是真实 contract 时用 `VARCHAR(n)`。
- money 或 exact decimal 用 `NUMERIC`；money 绝不用 binary float。
- 数据库必须 enforce 的 closed set 用 native PostgreSQL enum 或 check constraint。

使用 constraints：

```python
from sqlalchemy import CheckConstraint, UniqueConstraint


__table_args__ = (
    UniqueConstraint("tenant_id", "slug", name="uq_project_tenant_slug"),
    CheckConstraint("price_cents >= 0", name="ck_product_price_cents_non_negative"),
)
```

规则：

- 给 constraint 和重要 index 命名。
- 只给 join 和 predicate 中使用的 foreign key 加 index，不是每列都加。
- partial 和 expression index 只有匹配真实 query predicate 时才用。
- case-insensitive unique email 优先 normalized column 或 functional unique index。
- 每次 schema change 都用 migration。

## Sessions 和 Transactions

Async 默认：

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(settings.database_url, pool_pre_ping=True)
SessionFactory = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionFactory() as session:
        yield session
```

Unit of work：

```python
async def create_user(session: AsyncSession, data: UserCreate) -> User:
    user = User(email=str(data.email), name=data.name)
    session.add(user)
    await session.flush()
    return user
```

规则：

- 除非项目有清晰 unit-of-work abstraction，否则 caller 拥有 commit/rollback。
- service/job boundary 使用 `async with session.begin()`。
- commit 前需要 database-generated value 时使用 `flush()`。
- low-level helper 不要 commit，除非该 function 明确是完整 transaction boundary。
- 不要在 concurrent task 之间共享 session。
- async ORM 设置 `expire_on_commit=False`，避免意外 expired attribute access。

## Query Style

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


async def get_user_by_email(session: AsyncSession, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return await session.scalar(stmt)
```

规则：

- 使用 `select()`、`insert()`、`update()`、`delete()`；避免 legacy `Query`。
- 单个 scalar ORM entity 或 value 使用 `session.scalar(stmt)`。
- 多个 ORM entity 或 scalar value 使用 `session.scalars(stmt)`。
- row tuple/mapping 或 DML 需要时使用 `session.execute(stmt)`。
- 有意使用 `one()`、`one_or_none()`、`first()` 和 `all()`。
- 不要为了 count 加载 row；使用 `select(func.count())`。
- 避免 N+1 query；显式选择 loader option。
- 业务规则允许时，bulk work 使用 bulk `insert().values([...])` 或 set-based `update()`，不要 per-row loop。

## Relationship Loading

- collection relationship 优先 `selectinload`。
- 小 many-to-one relationship 在减少 round trip 时优先 `joinedload`。
- 避免 API serialization 期间意外 lazy loading。
- async code 尤其要显式，因为 lazy load 可能在预期 greenlet/session context 外失败。

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.posts)).where(User.id == user_id)
user = await session.scalar(stmt)
```

## Migrations

规则：

- 使用 Alembic 或项目现有 migration tool。
- migration 必须 reversible，除非项目有 documented forward-only policy。
- migration 包含 schema constraint、index 和 server default。
- 大表避免长时间 blocking operation；需要时使用 concurrent index 或 staged migration。
- production schema change 不依赖 `metadata.create_all()`。
- 应用 autogenerated migration 前先 review。Autogenerate 是草稿，不是真相来源。
- data migration 尽量 explicit 且 idempotent。

## Pydantic 边界

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr


async def read_user(session: AsyncSession, user_id: int) -> UserRead | None:
    user = await session.get(User, user_id)
    return None if user is None else UserRead.model_validate(user)
```

规则：

- 在边界把 ORM object 转成 Pydantic。
- 避免把 ORM object 传过不需要 persistence behavior 的 layer。
- 避免在 Pydantic validator 中 query database。
- 调 `model_validate` 前，query 先包含 serialization 所需内容。

## Concurrency 和 Locking

- race-prone uniqueness 用 unique constraint 加 retry/handled `IntegrityError`。
- 只有真实 critical section 需要时才用 `SELECT ... FOR UPDATE`。
- 事务保持短。
- job 和 webhook 明确 idempotency。
- lost update 重要时用 optimistic locking/version column。
- 不要在 concurrent task 之间共享一个 `AsyncSession`。
- outbox/event publishing 要在 external delivery 前，原子持久化 database change 和 event。

## 避免

- 新代码中的 `session.query(...)`。
- implicit autocommit 假设。
- 在 request handler 或 hot function 中创建 engine/session。
- import-time database connection。
- 每个 query 只包一层的宽泛 repository abstraction。
- 删除超过 caller 预期的 cascade。
- normal column 更清楚安全时仍使用 JSONB。
