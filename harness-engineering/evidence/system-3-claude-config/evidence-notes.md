# System 3 — E-Commerce Team Claude Code Config — Evidence Notes

**System path:** `Configure Claude Code for a Multi-Surface Monorepo Team/04-plan-mode-and-explore-decision-doc/solution/`
**Environment:** Python 3.12.10, dedicated venv (`C:\Users\<user>\venvs\hx3`), Windows 11 (no WSL)

## What was run

| Check | Command | Result |
|---|---|---|
| Test suite | `pytest tests/ -v` | **35 passed** (see `test-log.txt`) |
| Validator | `python -m ecommerce_team_config .` | **`OK`, exit code 0** (see `validator-output.txt`) |

No API key is required for this system.

## Key configuration facts (see `claude-structure.txt`)

- **`@import` hierarchy** — `CLAUDE.md` lines 21–24 pull in four modular standards files:
  `@.claude/standards/frontend.md`, `@.claude/standards/api.md`, `@.claude/standards/database.md`, `@.claude/standards/testing.md`.
- **Path-scoped rules** — three rule files under `.claude/rules/` declare glob patterns in YAML frontmatter (`paths:` key):
  - `api.md` → `src/api/**/*`
  - `react.md` → `src/components/**/*`, `src/pages/**/*`
  - `tests.md` → `**/*.test.tsx`, `**/*.test.ts`
- **Project-scoped slash command** — `.claude/commands/review.md` (`/review`), with `argument-hint` and a git/gh-restricted `allowed-tools` list.
- **Forked skill** — `.claude/skills/deploy-check/SKILL.md` declares `context: fork` and a read-only `allowed-tools` set (Read, Grep, Glob, plus read-only `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, `Bash(gh pr view:*)` — no write-capable tools).
- **Scope documentation** — `CLAUDE.md` line 41 contrasts project scope (this file + `.claude/standards/`, version-controlled and shared) with user scope (`~/.claude/CLAUDE.md`, "user-level, never shared"). The validator checks this via `distinguishes_project_vs_user_scope` in `ecommerce_team_config/claude_md.py`.

## Notes

- The test count observed (35) matches both the task list and the rubric.
- The validator is a deterministic enforcement point: any missing frontmatter glob, a non-fork skill context, or a write-capable tool in the skill allowlist makes it exit non-zero.
