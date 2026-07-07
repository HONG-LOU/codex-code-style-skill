# 交付风格

commit、push、tag、release 或 finalize code change 时，使用本参考。

## Git Workflow

1. 用 `git status --short --branch`、`git diff` 和 `git diff --cached` 检查改动。
2. 用显式路径只 stage 目标文件。
3. 运行适合当前项目的 repository validation。
4. validation 失败或改动文件时，最小修正、重新 stage、重跑相关检查。
5. 用 `git log --oneline -5` 检查近期 commit 风格。
6. 用符合仓库风格的简洁 message commit。
7. push 到 tracked remote branch。

## 安全

- 除非用户明确要求，不使用 destructive git command。
- 除非用户明确要求，不 amend、不 force push。
- commit 中不能有 secret；stage 前检查 diff。
- 尊重 dirty worktree。不要 revert 无关用户改动。
- Codex 会话中避免 interactive git command。

## Python 验证默认值

先用项目自己的命令。现代 Python service 的典型检查：

```powershell
uv run ruff format .
uv run ruff check . --fix
uv run pre-commit run --all-files
uv run mypy . --no-error-summary
uv run pytest
uv run ruff format --check --diff .
uv run ruff check .
```

迭代时跑较窄检查；shipping shared behavior 前再跑更宽检查。

## OTA Release Notes

- Dev image tag 通常保持 `latest`。
- Staging tag 通常用 `v<version>rc<N>`。
- Production tag 通常用 `v<version>`。
- 常规 OTA production release 优先 patch bump，除非 Eric 明确要求 minor、major 或 breaking release。
- OTA production 不要直接跑 `uv run cz bump --yes`，除非确实要 minor/major bump。

## Post-Ship Report

报告：

- Branch。
- Commit hash 和 message。
- 已运行检查及结果。
- Push 结果。
