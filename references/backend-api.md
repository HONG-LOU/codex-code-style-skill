# Backend And FastAPI API Style

Use this reference when changing FastAPI routers, API schemas, dependency injection, services, repositories, settings, error handling, middleware, OpenAPI docs, auth, or background work.

## Architecture

- Organize by project convention first. For FastAPI services, prefer feature/domain packages with clear boundaries.
- Keep routers thin: request parsing, dependencies, status codes, and response models.
- Put business rules, orchestration, authorization decisions, and transaction ownership in services.
- Keep repositories or query helpers focused on data access. They should not decide business outcomes.
- Use dependency injection by parameters and FastAPI `Depends`; do not instantiate services deep inside routers or other services unless the project pattern requires it.
- Inner layers must not import outer layers: services should not depend on FastAPI `Request`, `Response`, `HTTPException`, or router modules.

## API Design

- Use plural nouns for collections and stable lowercase paths. Follow existing path style before changing conventions.
- Use `GET` for reads, `POST` for create/actions, `PATCH` for partial updates, `DELETE` for deletion.
- Every route should declare a response type or `response_model` unless the project intentionally streams or returns raw responses.
- Return status codes explicitly for creates, deletes, accepted background work, and errors.
- Public list endpoints need bounded pagination. Default small, enforce hard max.
- Do not blindly wrap all responses in `{code, data}` unless the existing API contract requires it.
- Use RFC 7807-style error bodies where the project supports it: `detail`, `type`, `instance`.
- Keep decorators and params lean. Avoid `summary`, `description`, `response_description`, and parameter descriptions when names and types already explain the contract.
- Keep OpenAPI text only for non-obvious behavior: external protocol quirks, security constraints, deprecation, side effects, unusual filtering semantics, or client-visible compatibility requirements.

## Pydantic Schemas

- Use separate schemas for create, update/patch, filters, and response when fields or validation differ.
- Use `ConfigDict(from_attributes=True)` for response schemas built from ORM objects.
- Use `extra="forbid"` for strict client input unless backward/forward compatibility requires `ignore`.
- Use `model_dump(exclude_unset=True)` or `model_fields_set` for PATCH semantics.
- Never return ORM models directly from API routes.

## Errors And Security

- Define domain exceptions close to the domain. Map them to HTTP in global handlers.
- Routers should not catch service exceptions only to translate them.
- Check authorization before mutation or expensive work.
- Do not log secrets, tokens, full connection strings, or raw PII.
- Configure CORS tightly in production. Do not use wildcard origins with credentials.
- Add or preserve security headers where the app owns middleware: `X-Content-Type-Options`, `X-Frame-Options`, HSTS behind HTTPS, and request IDs.

## Settings And Background Work

- Use `pydantic-settings`; validate config at startup. Do not read env vars in business logic.
- Use FastAPI background tasks only for simple best-effort work.
- Use durable queues for retryable, long-running, or business-critical background jobs.
- Keep background jobs idempotent and observable.
