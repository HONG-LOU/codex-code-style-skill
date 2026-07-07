# Pydantic v2 风格

创建或修改 Pydantic model、validation、serialization、settings、DTO 或 API schema 时，使用本参考。

## 核心规则

- 只使用 Pydantic v2。
- 把 Pydantic 当作边界层：HTTP body、config、message、external API payload、CLI input 和 persistence DTO。
- ORM model、domain object 和 Pydantic schema 保持分离，除非项目有意非常小。
- 优先 constrained type 和 `Annotated` metadata，而不是 custom validator。
- model 保持小、可组合、用途明确：create/read/update/filter 通常应是不同 schema。
- 不要在 validator 或 serializer 里放 database I/O、network I/O 或 session access。

## Model Configuration

使用 `ConfigDict` 作为 `model_config`。

```python
from pydantic import BaseModel, ConfigDict


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True, extra="forbid")

    id: int
    email: str
```

默认：

- input model 默认 `extra="forbid"`，除非需要 forward-compatible payload。
- 从 ORM object 构造 DTO 时用 `from_attributes=True`。
- 用户输入字符串应 normalize 时用 `str_strip_whitespace=True`。
- 只有 mutable long-lived model 确实需要时才用 `validate_assignment=True`。
- 边界必须同时接受 field name 和 alias 时，用 `ConfigDict(validate_by_name=True, validate_by_alias=True)`。新代码不要用 `populate_by_name=True`。
- 依赖 alias output 的 API schema 显式设置 `serialize_by_alias`。
- config、ID、money、enum-like protocol 和 internal message 有意使用 `strict=True`。只有 user-facing input contract 要求时才允许 coercion。
- 避免 global alias 和 coercion knobs，除非 API contract 要求。

## Fields

字段要精确。

```python
from typing import Annotated

from pydantic import BaseModel, EmailStr, Field

NonEmptyName = Annotated[str, Field(min_length=1, max_length=120)]


class UserCreate(BaseModel):
    email: EmailStr
    name: NonEmptyName
    age: Annotated[int, Field(ge=13, le=130)]
```

规则：

- dynamic default 使用 `Field(default_factory=...)`。
- 只有 `None` 合法时才用 `default=None`。
- constrained default 是动态的或必须像用户输入一样检查时，用 `validate_default=True`。
- immutable value 有用时使用 `frozen=True`。
- secret 使用 `SecretStr` 或 `SecretBytes`。
- 只有外部 contract 确实在 JSON 内发送 encoded JSON 时，才用 `Json[...]`。
- min/max/regex/list length 避免 validator；使用 `Field` constraint。

## Validation

使用 v2 validator。

```python
from typing import Self

from pydantic import BaseModel, field_validator, model_validator


class PasswordChange(BaseModel):
    password: str
    confirm_password: str

    @field_validator("password")
    @classmethod
    def validate_password(cls, value: str) -> str:
        if len(value) < 12:
            raise ValueError("password must contain at least 12 characters")
        return value

    @model_validator(mode="after")
    def passwords_match(self) -> Self:
        if self.password != self.confirm_password:
            raise ValueError("passwords do not match")
        return self
```

规则：

- 单字段使用 `field_validator`。
- 跨字段 invariant 使用 `model_validator(mode="after")`。
- `mode="before"` 只用于 raw legacy payload normalization。
- 复用 scalar constraint 时，先用 `Annotated[..., Field(...)]` 或 reusable annotated validator，再考虑重复 model validator。
- validator 返回声明类型。
- validator 保持 deterministic、pure 和 cheap。
- 新代码不要用 `@validator`、`@root_validator`、`parse_obj`、`parse_raw`、`from_orm`、`.dict()` 或 `.json()`。

## Serialization

只有默认 serialization 不是 contract 时，才使用显式 serializer。

```python
from datetime import datetime

from pydantic import BaseModel, field_serializer


class EventRead(BaseModel):
    happened_at: datetime

    @field_serializer("happened_at")
    def serialize_happened_at(self, value: datetime) -> str:
        return value.isoformat()
```

规则：

- Python data 使用 `model_dump()`。
- JSON string 使用 `model_dump_json()`。
- PATCH/update payload 使用 `model_dump(exclude_unset=True)`。
- derived read-only field 使用 `computed_field`。
- 只有外部 contract 要求时传 `by_alias=True` 或设置 `serialize_by_alias=True`；内部 dump 保持 field-name based。
- 不要为了一个 endpoint 全局覆盖 serialization。

## TypeAdapter

standalone validation 使用 `TypeAdapter`，不要创建一次性 wrapper model。

```python
from pydantic import TypeAdapter

UserIds = TypeAdapter(list[int])

ids = UserIds.validate_python(raw_ids)
```

使用场景：

- 验证 `list[Model]`、`dict[str, Model]`、union、literal 和简单 constrained value。
- parse 外部 JSON array。
- 完整 `BaseModel` 没必要时，验证 `TypedDict` payload。
- 验证 settings fragment 或 message payload。

## ORM 边界

不要让 Pydantic schema 继承 SQLAlchemy model，也不要把 SQLAlchemy column 挂到 Pydantic field。

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr


user_read = UserRead.model_validate(user_orm)
```

规则：

- 只 select response 需要的字段。
- input schema 和 output schema 分开。
- ORM 到 schema 转换使用 `from_attributes=True`。
- 避免从 API boundary 返回 ORM entity。
- 避免 serialization 期间意外 lazy-load；显式塑造 query。

## Settings

应用设置使用 `pydantic-settings`。

```python
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: SecretStr
```

规则：

- settings 实践上保持 immutable；应用启动时创建一个实例。
- 不要在业务逻辑到处读 environment variable。
- secret 保持 secret；只有构造必须要 raw value 的 client 时才 reveal。

## Performance

- 在边界 validate 一次；避免 hot loop 中重复 `model_validate`。
- JSON bytes 或 string 优先 `model_validate_json()`，而不是 `json.loads()` 后接 `model_validate()`。
- 同类型重复验证时，在 module scope 复用 `TypeAdapter` instance。
- 除非需要，避免 wrap validator。
- large collection 只需要第一个错误时，考虑 fail-fast validation。
- model field 尽量使用 `list` 和 `dict` 这类 concrete container。
- `model_construct()` 只用于 profiling 后的 trusted data 或狭窄 internal adapter。

## Minimal Patterns

Create：

```python
class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: EmailStr
    name: Annotated[str, Field(min_length=1, max_length=120)]
```

Patch：

```python
class UserPatch(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    name: Annotated[str, Field(min_length=1, max_length=120)] | None = None

    def changes(self) -> dict[str, object]:
        return self.model_dump(exclude_unset=True)
```

Read：

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr
    name: str
```
