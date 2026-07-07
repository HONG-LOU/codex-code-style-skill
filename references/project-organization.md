# Project Organization

Use this reference when changing package layout, routers, services, repositories, dependency wiring, settings, scripts, docs, migrations, tests, CI, or root-level project config.

## Discovery And Scope

Before moving files or changing layers, inspect:

- Repository root files: `pyproject.toml`, lockfiles, tool configs, CI, README, docs, scripts, migrations, and ignore files.
- Import roots and package layout: `src/`, top-level app packages, namespace packages, app factories, and entrypoints.
- Existing layer names: `routers`, `api`, `services`, `domain`, `models`, `schemas`, `repositories`, `queries`, `clients`, `jobs`, `tasks`, `dependencies`, `settings`, `tests`.
- The touched behavior path: caller, callee, tests, schemas, database models, migrations, docs, and background or CLI entrypoints that share the same service logic.

Define the behavior-preserving boundary before refactoring. If a move changes public imports, API behavior, database semantics, auth, transaction ownership, or generated docs, treat it as a behavior change and test it explicitly.

## Repository Root

- Keep the root boring and scannable. It should mostly contain project metadata, lockfiles, docs entrypoints, CI/config files, `src/` or the app package, `tests/`, `docs/`, `migrations/` or `alembic/`, and focused `scripts/` or `tools/`.
- Prefer `pyproject.toml` as the primary home for Python tool config when the project already uses it or when starting fresh. Keep separate config files only when the tool or repo convention benefits from them.
- Commit lockfiles for applications and services. Do not hand-edit generated lockfiles.
- Keep generated artifacts, virtualenvs, local caches, coverage output, and build output out of the repo unless the project intentionally vendors them.
- Do not leave importable one-off modules at the repo root. Move reusable code into the package and operational scripts into `scripts/` or `tools/`.
- Prefer `src/` layout for new maintained packages or services that need packaging/import isolation. Do not migrate an existing flat layout only for style unless packaging or test imports are part of the problem.

## Package Layout

Choose the smallest structure that makes ownership clear:

- Tiny app: a compact package with `main.py`, `settings.py`, `models.py`, `schemas.py`, and focused modules is fine.
- Medium service: prefer feature or domain packages, with local `router.py`, `schemas.py`, `service.py`, `repository.py` or `queries.py`, `models.py`, `dependencies.py`, and `exceptions.py` only where each file has real content.
- Shared platform code: use explicit shared packages such as `auth`, `database`, `config`, `observability`, or `integrations`. Do not create vague cross-domain `utils.py` dumping grounds.

Use layer folders only when they reduce coupling. Do not create empty `services/`, `repositories/`, or `domain/` directories as architecture theater.

## Layer Responsibilities

### Entrypoints

Routers, CLI commands, background job handlers, event consumers, and webhook handlers are adapters. They should:

- Bind protocol details: path/body/query params, dependencies, auth context, status codes, response models, CLI args, message metadata.
- Validate and convert input at the boundary.
- Call one service or use-case function for the business operation.
- Return typed response schemas or exit results.

They should not:

- Execute business workflows, make persistence decisions, or own multi-step transactions.
- Build raw response dictionaries when a schema can express the output.
- Catch domain/service exceptions only to translate them locally when global handlers or caller-level mapping should own that.

### Services And Use Cases

Services own application behavior:

- Orchestrate domain rules, repositories, external clients, authorization decisions, idempotency, and transaction boundaries.
- Accept typed request/use-case data and dependencies as parameters.
- Return typed domain results or response DTOs, not FastAPI `Response`, `Request`, `HTTPException`, or router-only schemas.
- Keep transaction ownership explicit. Use a Unit of Work when one use case coordinates multiple repositories or writes.

Prefer module-level service functions when no durable state is needed. Use service classes only when they carry meaningful dependencies or match the project pattern.

### Domain

Domain modules contain rules that should be testable without HTTP, database sessions, environment variables, clocks, or network I/O. Use dataclasses, Pydantic models, value objects, enums, and pure functions where they make invalid states harder to express.

### Persistence

Repositories, query helpers, and data modules own database access:

- Accept a session, connection, or Unit of Work explicitly.
- Return ORM entities, row DTOs, or typed result objects according to project convention.
- Keep query shape, eager loading, locking, and persistence mapping visible.
- Do not decide business outcomes, read HTTP request state, import routers, or raise web exceptions.

Use a repository only when it hides real persistence complexity or gives services a clearer contract. A well-named query function is enough for simple read paths.

### External Adapters

HTTP clients, SDK wrappers, queues, file storage, email, payment, and AI clients belong behind focused adapter modules. Parse third-party payloads into typed boundary models before service logic sees them.

## Import Direction

Keep dependencies flowing inward:

```text
router / cli / job / event adapter
  -> service / use case
  -> domain
  -> repository / query / external adapter interfaces
```

Composition roots wire concrete dependencies: app factory, dependency providers, worker startup, CLI main, or test fixtures.

Forbidden unless the existing project has a deliberate exception:

- Services importing FastAPI router modules, `Request`, `Response`, `Depends`, or `HTTPException`.
- Domain code importing ORM sessions, web frameworks, environment config, or network clients.
- Repositories importing services, routers, API schemas, or auth dependencies.
- Shared modules importing feature modules just to avoid passing parameters.

## Config, Scripts, And Docs

- Put runtime settings in one typed settings module, preferably with `pydantic-settings`. Environment variables feed settings at startup; business logic receives typed config or parameters.
- Keep `.env.example` or documented env tables current when settings change. Never commit real secrets.
- Keep operational scripts in `scripts/` or `tools/`. Give reusable scripts a typed `main()` and no import-time side effects.
- Keep migrations in the project migration directory with descriptive names. Review generated migrations and avoid mixing unrelated schema changes.
- Use README as the project entry point: what the project is, how to install, configure, run, test, and deploy locally.
- Put durable docs in `docs/`. Separate tutorials, how-to guides, reference, and explanation when docs grow. Do not let docs become a second source of truth for behavior that code or tests already define.
- Record high-impact architecture decisions in short ADR-style notes only when the decision is stable and future maintainers need the rationale.

## Tests

Mirror behavior boundaries:

- Router/API tests verify HTTP contracts, auth, status codes, serialization, and error mapping.
- Service tests verify use-case behavior, domain decisions, transaction orchestration, and external adapter boundaries.
- Repository/database tests verify constraints, query shape, locks, migrations, and Postgres-specific behavior.
- CLI/job/event tests verify adapter parsing and delegation to services, not duplicated business logic.

When moving logic out of a router or script, move or add tests at the layer that now owns the behavior. Keep a small integration test at the entrypoint if the route/job/CLI contract matters.

## Existing Code Cleanup

When you see an organization smell in touched code:

1. Identify the current owner of the behavior.
2. Check immediate callers, callees, tests, schemas, and docs.
3. Move the smallest coherent block to the nearest existing layer.
4. Update imports and tests in the same behavior path.
5. Run focused tests first, then broader checks if imports or shared modules changed.

Defer the cleanup and mention it as follow-up when it requires cross-domain moves, public import changes, database behavior changes, large package reshuffles, or missing tests.

## Smell Checklist

Fix when it is in the touched path and verifiable:

- Router performs multi-step business decisions, direct SQL, transaction ownership, or external workflow orchestration.
- Dependency function mutates business state or performs expensive workflow logic.
- Service imports FastAPI, response schemas that exist only for HTTP, CLI parser types, or job framework objects.
- Repository imports services, routers, auth dependencies, API schemas, or returns raw dictionaries to business logic.
- Domain logic reads environment variables, clocks, network, database sessions, or global mutable state directly.
- Same use case is duplicated in API, CLI, job, and event handlers.
- `utils.py` or `helpers.py` mixes unrelated domains.
- Root has loose app modules, one-off scripts, duplicate config files, stale generated files, or undocumented required env vars.
- Tests only cover the entrypoint while service/domain behavior has become complex.

## Source Basis

These rules are informed by:

- FastAPI bigger applications and APIRouter organization: https://fastapi.tiangolo.com/tutorial/bigger-applications/
- Python Packaging `src` vs flat layout guidance: https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/
- Architecture Patterns with Python service layer, repository, and Unit of Work patterns: https://www.cosmicpython.com/book/chapter_04_service_layer.html
- Twelve-Factor config guidance: https://12factor.net/config
- pytest good integration practices and test layout guidance: https://docs.pytest.org/en/stable/explanation/goodpractices.html
- Ruff configuration guidance: https://docs.astral.sh/ruff/configuration/
- uv project configuration guidance: https://docs.astral.sh/uv/concepts/projects/config/
- Diataxis documentation framework: https://diataxis.fr/
