# CLAUDE.md — global-skills

This repo is a **data-store** — no application code. It holds machine-wide
Claude Code + Codex CLI skills that are symlinked to `~/.claude/skills` and
`~/.codex/skills/` on each workstation.

## Contents

- `conventional-commit/` — Conventional Commits + gitmoji workflow (global skill)
- `cross-agent-delegate/` — Cross-agent delegation guidance (global skill)
- `skill-creator/` — Skill authoring and evaluation (global skill)

## How skills are consumed

Skills in this repo are available machine-wide via symlinks managed by
`stromy-org/scripts/agent-sync.py`. Do not modify the symlink structure here;
run `agent-sync.py` from the stromy-org control plane to reconcile.

## Editing skills

Edit skill files directly. Each skill is a directory with a required
`SKILL.md` (frontmatter: `name`, `description`). Keep descriptions under
1024 characters.

## Commits

Use `/conventional-commit` for all commits (inherited globally).
