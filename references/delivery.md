# Delivery Style

Use this reference when committing, pushing, tagging, releasing, or finalizing code changes.

## Git Workflow

1. Inspect changes with `git status --short --branch`, `git diff`, and `git diff --cached`.
2. Stage only intended files with explicit paths.
3. Run repository validation appropriate to the project.
4. If validation fails or changes files, fix minimally, restage, and rerun relevant checks.
5. Check recent commit style with `git log --oneline -5`.
6. Commit with a concise message that matches the repository.
7. Push to the tracked remote branch.

## Safety

- No destructive git commands unless explicitly requested.
- No amend or force push unless explicitly requested.
- No secrets in commits; inspect diffs before staging.
- Respect dirty worktrees. Do not revert unrelated user changes.
- Avoid interactive git commands in Codex sessions.

## Python Validation Defaults

Use the project's own commands first. For modern Python services, typical checks are:

```powershell
uv run ruff format .
uv run ruff check . --fix
uv run pre-commit run --all-files
uv run mypy . --no-error-summary
uv run pytest
uv run ruff format --check --diff .
uv run ruff check .
```

Run narrower checks during iteration, then broader checks before shipping shared behavior.

## OTA Release Notes

- Dev image tags usually stay `latest`.
- Staging tags usually use `v<version>rc<N>`.
- Production tags usually use `v<version>`.
- For normal OTA production releases, prefer patch bumps unless Eric explicitly asks for minor, major, or breaking release.
- Do not run bare `uv run cz bump --yes` for OTA production unless a minor/major bump is intended.

## Post-Ship Report

Report:

- Branch.
- Commit hash and message.
- Checks run and result.
- Push result.
