---
name: code-style
description: Use when writing, editing, reviewing, refactoring, testing, committing, pushing, or shipping Python code, project structure, or repo configuration, especially Pydantic v2, SQLAlchemy 2.x, PostgreSQL, FastAPI, routers, services, repositories, package layout, docs, tooling, backend architecture, performance, pytest, uv, modern Python typing, file slimming, adjacent cleanup, or avoiding Any, raw dicts, misplaced logic, dead code, and unnecessary FastAPI descriptions.
---

# Code Style

Apply this skill whenever touching Python code, project layout, repo config, tests, or technical docs. Optimize for correctness, small code, explicit contracts, boring reliability, and compatibility with the current project.

Core principle: code and files must be typed, short, direct, reusable, and obvious. Project structure should make boundaries visible: HTTP entrypoints, business use cases, domain rules, persistence, external adapters, tests, config, and docs each belong in the smallest clear home. If raw data, dynamic access, manual conversion, or misplaced logic reaches business logic, push validation, normalization, and orchestration back to the right boundary.

## Operating Rules

1. Read the local project first: `pyproject.toml`, lockfiles, lint/type/test config, existing package layout, representative modules, docs, and tests.
2. Follow existing working patterns unless they conflict with correctness, security, modern APIs, or the user's explicit request.
3. Prefer the smallest change that fully solves the task. Do not introduce frameworks, base classes, services, repositories, factories, metaclasses, decorators, or middleware unless they remove real complexity now.
4. Make invalid states unrepresentable with types, constrained models, database constraints, and focused functions.
5. Keep public boundaries strict and explicit; keep internals simple.
6. Avoid compatibility with obsolete Python or library versions unless the project requires it.
7. Convert raw external data to typed objects at the edge. Do not pass raw dictionaries through service, builder, repository, or domain logic.
8. Inspect the immediate upstream and downstream of touched code: callers, callees, nearby tests, shared schemas, sibling modules, package boundaries, config, docs, and repeated local patterns.
9. Improve adjacent code or file organization when it is directly related, low risk, and verifiable. Do not expand into unrelated rewrites.
10. Define the behavior-preserving boundary before refactoring. Behavior changes must be explicit, requested or required, and tested.
11. Delete useless code, comments, metadata, whitespace, and characters whenever you touch nearby code.
12. Verify with the narrowest meaningful command first, then broader checks when the change has wider blast radius.

## Modern Baseline

Prefer these defaults when the project does not already define versions:

- Python: target 3.12+ syntax; use 3.13/3.14 features only when the project declares them.
- Pydantic: v2 APIs only.
- SQLAlchemy: 2.x typed ORM and Core APIs only.
- PostgreSQL: use native database constraints and types; do not write app-only integrity rules when the database can enforce them.
- Tooling: `uv`, `ruff`, `pyright` or strict `mypy`, `pytest`, `pytest-asyncio` where async tests exist.
- CI hygiene: locked installs, type checking, tests, and dependency/security checks for maintained services.

Do not churn a project just to update syntax. Modernize only touched code or code needed for the task.

## Project Organization

Read [references/project-organization.md](references/project-organization.md) before changing package layout, routers, services, repositories, dependencies, settings, scripts, docs, migrations, tests, CI, or root-level config.

Hard defaults:

- Make the architecture legible from the tree. Prefer clear feature/domain packages for non-trivial services; keep tiny projects simple until extra layers remove real complexity.
- Keep routers and other entrypoints thin. Move business decisions, orchestration, transactions, and external workflow logic to services or use-case functions when touching that behavior path.
- Keep import direction one way: entrypoint/adapters -> services/use cases -> domain -> persistence/external adapters. Inner layers must not import FastAPI, CLI, job, or router modules.
- Put persistence code in repositories, query helpers, or data modules; keep them free of business decisions and HTTP-specific types.
- Keep repo root boring: package code, tests, docs, migrations, scripts, lockfiles, and tool config should have obvious homes. Do not leave random importable modules, throwaway scripts, duplicate configs, or stale docs near touched work.
- Improve existing organization in the touched path and its immediate upstream/downstream. Rename, move, or split files only when behavior is covered or the change is mechanical and easy to verify.
- Do not perform broad architecture migrations, layer creation, `src/` migrations, package reshuffles, or docs rewrites unless the task requires them and tests make the blast radius acceptable.

## Code Shape

- Write functions that do one useful thing and return typed values.
- Prefer module-level functions over classes when no durable state or polymorphism is needed.
- Prefer small dataclasses or Pydantic models for structured data; avoid untyped dicts at boundaries.
- Use dependency injection by parameters. Avoid global mutable state, implicit singletons, hidden I/O, and import-time work.
- Keep names precise and boring: verbs for functions, nouns for data, booleans with `is_`, `has_`, `should_`, or `allow_`.
- Avoid clever one-liners when they hide error handling, queries, validation, or transaction semantics.

## File Slimming

- Keep files as small as the behavior allows. Remove dead code, unreachable branches, unused imports, unused variables, duplicate helpers, commented-out code, blank-line noise, and decorative separators.
- Comments are code weight. Delete them by default. Keep a comment only for non-obvious invariants, transactional reasoning, security constraints, external protocol quirks, or a documented limitation with a clear upgrade trigger.
- Do not write comments that restate names, types, control flow, route purpose, query purpose, or the next line. Improve the code name or shape instead.
- Do not add FastAPI `description`, `summary`, `response_description`, parameter descriptions, or long docstrings for obvious CRUD endpoints and simple query params. Keep them only when they document a non-obvious contract, external protocol, security behavior, deprecation, or generated API requirement.
- Delete nearby useless comments and metadata when you see them in the same touched behavior path. This is expected cleanup, not optional polish.
- Preserve public behavior. Removing comments, unused code, or redundant metadata must not change API contracts that clients depend on, generated docs that are explicitly required, or validation behavior.

## Expressive Minimalism

- Prefer one clear expression over two mechanical lines. Do not introduce temporary variables, helper functions, classes, or comments unless they clarify a real concept.
- Prefer declarative models and type constraints over imperative validation code. A Pydantic field constraint is better than repeated `if` checks; a serializer is better than hand-built output dictionaries.
- Use modern Python syntax when it reduces noise: pattern matching, assignment expressions in small local scopes, comprehensions, generator expressions, `StrEnum`, `Self`, PEP 695 generics, and `type` aliases.
- Keep one-liners only when they remain obvious. Split code when the line contains branching, error handling, transactions, queries, or multiple business concepts.
- Reuse through small typed functions, Pydantic models, protocols, and focused query helpers. Do not create vague `utils` modules or abstract base classes for one caller.
- When code repeats field mapping, normalization, serialization, or coercion, first look for a Pydantic model, `TypeAdapter`, validator, serializer, computed field, or typed helper that deletes the repetition.
- Before writing custom code, check whether the standard library, framework, current project helper, or installed dependency already solves it cleanly.

## Adjacent Code Cleanup

- Treat every touched area as a small maintenance window. Check the caller path, callee path, nearby tests, sibling modules, and shared schemas for the same smell you are already fixing.
- Fix adjacent issues when they are in the same behavior path and can be verified with the same or slightly broader tests: misplaced router logic, duplicated conversion, stale names, raw dict leakage, missing boundary validation, unnecessary comments, dead branches, unused code, redundant FastAPI metadata, inconsistent Pydantic usage, or stale docs/config.
- Keep behavior stable. Do not change database semantics, API contracts, auth behavior, transaction boundaries, query cardinality, or error semantics as incidental cleanup.
- Do not perform broad renames, architecture migrations, repository rewrites, or cross-cutting style churn unless the task requires it or tests make the blast radius acceptable.
- If the adjacent cleanup is valuable but risky, leave it out of the patch and mention it as follow-up instead of bundling it silently.

## Backend And API

Read [references/backend-api.md](references/backend-api.md) before changing FastAPI routers, request/response schemas, dependency injection, service boundaries, settings, auth, error handling, middleware, background tasks, or API documentation.

Hard defaults:

- Keep routers thin: HTTP binding, dependency injection, status codes, and response models only.
- Put business orchestration and transaction ownership in services. Keep repositories/data helpers free of business decisions.
- Use Pydantic request, response, create, update, filter, and settings models instead of raw dictionaries or ad hoc params.
- Use domain exceptions plus global handlers. Routers should not catch service exceptions just to turn them into HTTP responses.
- Use explicit `response_model`, status codes, pagination limits, auth checks, and security headers where the project owns the API surface.
- Keep FastAPI route decorators and params lean. Avoid obvious `description` and `summary` text.

## Performance And Concurrency

Read [references/performance-concurrency.md](references/performance-concurrency.md) before changing list endpoints, database queries, relationship loading, background work, caching, locking, idempotency, connection pools, high-concurrency flows, or slow paths.

Hard defaults:

- Avoid N+1 queries and accidental async lazy loading. Shape queries with eager loading or column selection before serialization.
- Cap public list endpoints. Prefer cursor/keyset pagination for large or append-only feeds.
- Keep transactions short. Use row-level locks, optimistic locking, unique constraints, or idempotency keys when concurrency can duplicate or lose writes.
- Offload slow external I/O and heavy CPU work to background jobs when request latency or retries matter.

## Typing Rules

- Type every new or changed function, method, class attribute, model field, module-level variable, and return value. Public boundaries must always be explicit.
- Use built-in generics: `list[str]`, `dict[str, int]`, `tuple[...]`, `set[...]`.
- Do not use bare `dict`, `list`, `tuple`, `set`, or `frozenset` annotations.
- Use `X | None`; avoid `Optional[X]` in new code.
- Use `Literal`, `Protocol`, `TypedDict`, `Self`, `Final`, `NewType`, and `override` when they make contracts clearer.
- Prefer `collections.abc` for parameter types: `Mapping`, `Sequence`, `Iterable`, `Callable`.
- Avoid `Any`. If unavoidable for opaque third-party blobs, performance-sensitive adapters, or intentionally unvalidated passthrough data, confine it to the smallest boundary and document the validation or trust boundary. Use `object` for unknown external values only at boundaries, never as a business-logic shortcut.
- Prefer PEP 695 syntax for new generic code when the project targets Python 3.12+: `class Repo[T]:` and `type Row = Mapping[str, object]`.
- Use `TypedDict` for narrow external payload shapes when a full Pydantic model would add noise; validate at the boundary before business logic.
- Do not use stringly typed statuses; use `StrEnum`, `Literal`, or constrained database enums/checks.
- Do not weaken types to satisfy tests; fix the production contract or test setup.

## Dicts And Dynamic Access

- Do not use raw dictionaries as structured data in business logic. Parse raw JSON, XML/SOAP params, Redis hashes, form payloads, and HTTP responses into Pydantic models or other typed structures immediately.
- Do not use `dict.get()` to access business fields. It silently hides typos and missing data. Use typed attributes after validation.
- Keep valid dictionary use narrow: HTTP headers, query params, simple lookup tables, typed local maps, external raw payloads before immediate validation, or SQL row mappings in a small local scope.
- Do not use `getattr`, `setattr`, or `hasattr` with constant attribute names. Use direct attribute access.
- Avoid updating ORM objects or DTOs by looping over `model_dump().items()` and calling `setattr`. Assign fields explicitly so migrations, invariants, and falsy values are visible in code.
- For dynamic dispatch, prefer `match`/`case`, a typed mapping of callables, or an explicit strategy object. Do not discover methods with `hasattr` plus `getattr`.
- Do not call `model_dump()` just to read keys inside business logic. Use model attributes. Reserve dumps for serialization, logging-safe summaries, external APIs, or PATCH diff helpers.

## Error Handling

- Fail fast at boundaries. Validate input once, then trust typed internal data.
- Raise narrow domain exceptions or standard exceptions with actionable messages.
- Do not catch broad exceptions unless adding context and re-raising, handling a real fallback, or cleaning up.
- Never silently coerce corrupt data. Use explicit conversion methods and tests.
- Keep logging structured and useful. Do not log secrets, tokens, full connection strings, or raw PII.

## Pydantic

Read [references/pydantic-v2.md](references/pydantic-v2.md) before creating or changing Pydantic models, validators, serializers, settings, DTOs, API schemas, or validation-heavy code.

Hard defaults:

- Use `BaseModel`, `ConfigDict`, `Field`, `Annotated`, `field_validator`, `model_validator`, `field_serializer`, `model_serializer`, `computed_field`, and `TypeAdapter`.
- Use `model_validate`, `model_validate_json`, `model_dump`, and `model_dump_json`; do not use v1 APIs such as `parse_obj`, `dict`, `json`, `from_orm`, `@validator`, or `@root_validator`.
- Prefer `Annotated[..., Field(...)]` constraints, standard constrained types, and discriminated unions over hand-written validation branches.
- Use `AliasChoices` plus `ConfigDict(validate_by_name=True, validate_by_alias=True)` for boundary models that must accept camelCase, snake_case, PascalCase, or legacy field names. Do not use `populate_by_name=True` in new code.
- Set `serialize_by_alias` explicitly on API schemas that rely on alias output; do not depend on Pydantic defaults changing across major versions.
- Choose strictness deliberately: use `strict=True` for configs, IDs, money, enum-like protocols, and internal messages; allow explicit coercion only at user-facing boundaries where it is part of the contract.
- Prefer `model_validate_json()` for JSON bytes or strings instead of `json.loads()` followed by `model_validate()`.
- Use `validate_default=True` when constrained defaults come from dynamic values or must be checked like user input.
- Set `extra="ignore"` on permissive external input models when upstream systems send irrelevant fields; use stricter handling where rejecting unknown fields is part of the contract.
- Use `model_fields_set` for PATCH or partial-update flows that must distinguish omitted fields from explicitly provided falsy values.
- Use `TypeAdapter` for standalone validation of lists, unions, primitives, and external payload fragments. Do not create wrapper models only to call validation.
- Use `computed_field`, serializers, aliases, and `model_dump` options to produce API shapes. Do not assemble response dictionaries field by field when a schema can express it.
- Use validators for boundary coercion and normalization. After validation, business logic should use attributes with already-correct types.
- Use `model_copy(update=...)` for typed immutable-style updates when it is clearer than mutating or round-tripping through dictionaries.
- Keep Pydantic schemas separate from SQLAlchemy ORM models.
- Use Pydantic at I/O boundaries, not as a replacement for every internal data object.

## SQLAlchemy And Postgres

Read [references/sqlalchemy-postgres.md](references/sqlalchemy-postgres.md) before changing ORM models, migrations, sessions, transactions, query code, or Postgres-specific behavior.

Hard defaults:

- Use SQLAlchemy 2.x style: `DeclarativeBase`, `Mapped[...]`, `mapped_column`, typed `relationship`, `select()`, `Session.execute`, `AsyncSession`, and `async_sessionmaker`.
- Never use legacy `Query`, implicit autocommit patterns, or untyped ORM attributes in new code.
- Keep one clear transaction owner per unit of work.
- Enforce integrity in Postgres with primary keys, foreign keys, unique constraints, check constraints, not-null constraints, indexes, and migrations.
- Use native Postgres types for real domain semantics: `UUID`, `JSONB`, `ARRAY`, `DateTime(timezone=True)`, `Text`, native enums/checks, and `Numeric` for money.
- Use Alembic or the project migration tool for every schema change; review autogenerated migrations and verify upgrade/downgrade where the project supports it.
- Do not mix ORM entities and API response schemas.
- When applying Pydantic data to ORM entities, assign explicit fields. For PATCH updates, use `model_fields_set` or explicit sentinel logic so valid falsy values are not lost.

## Tooling And Tests

Read [references/tooling-tests.md](references/tooling-tests.md) before adding or changing project tooling, lint/type config, tests, fixtures, or CI checks.

Defaults:

- Use `ruff format` and `ruff check --fix` if the project uses Ruff.
- Prefer Ruff rules that catch dynamic attribute mistakes and avoidable complexity, including `B009` and `B010` where practical.
- Use the project's existing type checker. Prefer strictness for changed code.
- Use `pytest` with focused unit tests for pure logic and integration tests for database behavior.
- Test behavior, edge cases, validation failures, API contracts, authorization, idempotency, transaction rollback paths, and query shape when relevant.
- Mock external HTTP with purpose-built transports or libraries such as `httpx.MockTransport` or `respx`. Do not hit real third-party services in unit tests.

## Delivery

Read [references/delivery.md](references/delivery.md) before committing, pushing, tagging, releasing, or finalizing code changes.

Hard defaults:

- Inspect `git status --short --branch`, `git diff`, and staged diff before committing.
- Stage only intended files. Do not include unrelated user changes.
- Run repository validation before commit. If checks change files or fail, fix minimally and rerun the relevant checks.
- Never use destructive git commands, amend, force push, or interactive git flows unless the user explicitly asks.
- Report branch, commit hash/message, validation commands, and push result after shipping.

## Decision Heuristics

- Prefer deleting code over abstracting it.
- Prefer explicit parameters over hidden configuration.
- Prefer a well-named query function over a generic repository layer unless the project already uses repositories.
- Prefer database constraints over duplicated service checks.
- Prefer `TypeAdapter` over wrapper models for standalone validation.
- Prefer `Annotated` constraints over validators for simple field rules.
- Prefer `model_validator(mode="after")` for cross-field invariants.
- Prefer `computed_field` or serializers over ad hoc response assembly.
- Prefer `AliasChoices` over manual key normalization for inbound API payloads.
- Prefer explicit attribute assignment over loops that copy dictionaries into objects.
- Prefer `model_fields_set` over truthiness checks for partial updates where `0`, `False`, or `""` may be valid values.
- Prefer improving touched callers, callees, and tests over leaving nearby directly related rot in place.
- Prefer moving misplaced behavior to the nearest existing layer over inventing a new architecture pattern.
- Prefer `selectinload` for collection relationships and explicit eager loading over accidental lazy loads.
- Prefer `datetime` with timezone awareness for persisted timestamps.
- Prefer structured concurrency (`asyncio.TaskGroup`) over bare `asyncio.gather` when failures must cancel siblings or propagate predictably.
- Prefer pydantic-settings for application settings. Do not call `os.getenv()` throughout business logic.
- Prefer domain exceptions, typed results, and global handlers over scattered HTTP-specific catches.

## Avoid

- Pydantic v1 APIs.
- `populate_by_name=True` in new Pydantic models.
- SQLAlchemy legacy ORM APIs.
- `Any`, bare collection annotations, and `object` in business-logic signatures.
- `dict.get()` for business fields.
- `getattr`, `setattr`, or `hasattr` with constant attribute names.
- `model_dump()` followed by dict subscripting or generic `setattr` updates.
- Raw dictionaries crossing service, builder, repository, or domain boundaries.
- Verbose glue code that Pydantic fields, validators, serializers, computed fields, or `TypeAdapter` can replace.
- Manual response dictionary assembly when a schema can express the output shape.
- Useless comments, commented-out code, dead code, decorative separators, and blank-line noise.
- Obvious FastAPI `summary`, `description`, `response_description`, or parameter descriptions.
- Premature architecture layers.
- Misplaced business logic in routers, dependencies, repositories, migrations, scripts, config modules, or tests.
- Inner-layer imports from FastAPI routers, CLI/job entrypoints, or web-only schemas.
- Root-level loose modules, duplicate config files, stale docs, and vague cross-domain `utils.py` modules.
- Broad incidental rewrites that are not covered by tests or required for the current behavior path.
- Broad `except Exception`.
- `os.getenv()` in business logic.
- Real network calls in unit tests.
- Unbounded public list endpoints and implicit relationship lazy loads.
- Runtime monkeypatching for application design.
- Mutable default arguments.
- Dynamic attributes and magic strings where typed structures fit.
- Large helper modules with vague names like `utils.py` unless the project already has focused conventions.
- Test-only changes that mask a production bug.

## Enforcement Checklist

Before finishing Python work, confirm the diff has:

- Zero new `Any` except tightly validated adapter boundaries.
- Zero new bare collection annotations.
- Zero new `dict.get()` for business fields.
- Zero new constant-name `getattr`, `setattr`, or `hasattr`.
- Zero new `model_dump()` plus dict subscript access in business logic.
- Zero new `populate_by_name=True`; use `validate_by_name=True` and `validate_by_alias=True` where needed.
- API schemas with alias output set serialization behavior explicitly.
- Raw external data parsed into typed structures at the boundary.
- Explicit ORM updates, especially for PATCH flows.
- Pydantic or type-system features used to remove avoidable boilerplate without hiding behavior.
- Immediate callers, callees, nearby tests, and sibling patterns checked for directly related cleanup.
- Adjacent refactors limited to behavior-preserving, verifiable changes.
- Project organization issues in the touched path checked: misplaced logic, wrong layer imports, stale docs/config, vague helper modules, and root clutter.
- API/router changes keep HTTP concerns separate from business logic.
- Query/list changes consider pagination, eager loading, indexes, and transaction scope.
- Files are slimmer: no new dead code, commented-out code, useless comments, decorative separators, or obvious FastAPI descriptions.
- Project lint, format, type, and test commands passing at the narrowest meaningful scope.
- Delivery work inspects diffs, stages only intended files, validates before commit, and pushes to the tracked remote.
