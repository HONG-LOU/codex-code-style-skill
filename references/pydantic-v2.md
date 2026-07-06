# Pydantic v2 Style

Use this reference when creating or changing Pydantic models, validation, serialization, settings, DTOs, or API schemas.

## Core Rules

- Use Pydantic v2 only.
- Treat Pydantic as a boundary layer: HTTP bodies, configs, messages, external API payloads, CLI inputs, and persistence DTOs.
- Keep ORM models, domain objects, and Pydantic schemas separate unless the project is intentionally tiny.
- Prefer constrained types and `Annotated` metadata over custom validators.
- Keep models small, composable, and purpose-specific: create/read/update filters should usually be separate schemas.
- Do not put database I/O, network I/O, or session access inside validators or serializers.

## Model Configuration

Use `ConfigDict` as `model_config`.

```python
from pydantic import BaseModel, ConfigDict


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True, extra="forbid")

    id: int
    email: str
```

Defaults:

- `extra="forbid"` for input models unless forward-compatible payloads are required.
- `from_attributes=True` for DTOs built from ORM objects.
- `str_strip_whitespace=True` when user-entered strings should be normalized.
- `validate_assignment=True` only for mutable long-lived models that need it.
- Use `ConfigDict(validate_by_name=True, validate_by_alias=True)` when a boundary must accept both field names and aliases. Do not use `populate_by_name=True` in new code.
- Set `serialize_by_alias` explicitly on API schemas that depend on alias output.
- Use `strict=True` deliberately for configs, IDs, money, enum-like protocols, and internal messages. Allow coercion only where user-facing input contracts require it.
- Avoid global aliases and coercion knobs unless the API contract requires them.

## Fields

Prefer precise fields.

```python
from typing import Annotated

from pydantic import BaseModel, EmailStr, Field

NonEmptyName = Annotated[str, Field(min_length=1, max_length=120)]


class UserCreate(BaseModel):
    email: EmailStr
    name: NonEmptyName
    age: Annotated[int, Field(ge=13, le=130)]
```

Rules:

- Use `Field(default_factory=...)` for dynamic defaults.
- Use `default=None` only when `None` is valid.
- Use `validate_default=True` when constrained defaults are dynamic or must be checked like user input.
- Use `frozen=True` for immutable values when useful.
- Use `SecretStr` or `SecretBytes` for secrets.
- Use `Json[...]` only when the external contract truly sends encoded JSON inside JSON.
- Avoid validators for min/max/regex/list length; use `Field` constraints.

## Validation

Use v2 validators.

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

Rules:

- Use `field_validator` for one field.
- Use `model_validator(mode="after")` for cross-field invariants.
- Use `mode="before"` only for raw legacy payload normalization.
- Use `Annotated[..., Field(...)]` or reusable annotated validators for reusable scalar constraints before adding repeated model validators.
- Return the declared type from validators.
- Keep validators deterministic, pure, and cheap.
- Do not use `@validator`, `@root_validator`, `parse_obj`, `parse_raw`, `from_orm`, `.dict()`, or `.json()` in new code.

## Serialization

Use explicit serializers only when default serialization is not the contract.

```python
from datetime import datetime

from pydantic import BaseModel, field_serializer


class EventRead(BaseModel):
    happened_at: datetime

    @field_serializer("happened_at")
    def serialize_happened_at(self, value: datetime) -> str:
        return value.isoformat()
```

Rules:

- Use `model_dump()` for Python data.
- Use `model_dump_json()` for JSON strings.
- Use `model_dump(exclude_unset=True)` for PATCH/update payloads.
- Use `computed_field` for derived read-only fields.
- Pass `by_alias=True` or set `serialize_by_alias=True` only when that is the external contract; keep internal dumps field-name based.
- Avoid overriding serialization globally for one endpoint.

## TypeAdapter

Use `TypeAdapter` for standalone validation instead of creating throwaway wrapper models.

```python
from pydantic import TypeAdapter

UserIds = TypeAdapter(list[int])

ids = UserIds.validate_python(raw_ids)
```

Use cases:

- Validating `list[Model]`, `dict[str, Model]`, unions, literals, and simple constrained values.
- Parsing external JSON arrays.
- Validating `TypedDict` payloads where a full `BaseModel` would be unnecessary.
- Validating settings fragments or message payloads.

## ORM Boundary

Do not inherit Pydantic schemas from SQLAlchemy models or attach SQLAlchemy columns to Pydantic fields.

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr


user_read = UserRead.model_validate(user_orm)
```

Rules:

- Select only the fields needed by the response.
- Use separate input and output schemas.
- Use `from_attributes=True` for ORM to schema conversion.
- Avoid returning ORM entities from API boundaries.
- Avoid lazy-load surprises during serialization; shape queries explicitly.

## Settings

Use `pydantic-settings` for application settings.

```python
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: SecretStr
```

Rules:

- Keep settings immutable in practice; create one instance at application startup.
- Do not read environment variables throughout business logic.
- Keep secrets as secrets; reveal only when constructing a client that requires the raw value.

## Performance

- Validate once at boundaries; avoid repeated `model_validate` inside hot loops.
- Prefer `model_validate_json()` for JSON bytes or strings instead of `json.loads()` followed by `model_validate()`.
- Reuse `TypeAdapter` instances at module scope when validating the same type repeatedly.
- Avoid wrap validators unless needed.
- Consider fail-fast validation for large collections when the first error is enough.
- Prefer concrete containers like `list` and `dict` in model fields when possible.
- Use `model_construct()` only for trusted data after profiling or in narrow internal adapters.

## Minimal Patterns

Create:

```python
class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: EmailStr
    name: Annotated[str, Field(min_length=1, max_length=120)]
```

Patch:

```python
class UserPatch(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    name: Annotated[str, Field(min_length=1, max_length=120)] | None = None

    def changes(self) -> dict[str, object]:
        return self.model_dump(exclude_unset=True)
```

Read:

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr
    name: str
```
