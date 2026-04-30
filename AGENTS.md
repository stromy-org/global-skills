# AGENTS.md — global-skills

Self-contained instruction file for Codex CLI, Gemini CLI, and any AI agent
working in this repository.

## What this repo is

`global-skills` is a **data-store** repository. It contains machine-wide AI
agent skills that are symlinked to `~/.claude/skills`, `~/.codex/skills/`, and
`~/.gemini/skills/` on each developer workstation. There is no application
source code here.

## Contents

| Directory | Purpose |
|-----------|---------|
| `conventional-commit/` | Conventional Commits + gitmoji workflow — global skill available in every repo |
| `cross-agent-delegate/` | Guidance for delegating work between Claude Code, Codex CLI, and Gemini CLI |
| `skill-creator/` | Skill authoring, evaluation, and optimization |

## Key rules for this repo

- **No application code** — only skill instruction files (`SKILL.md`,
  `references/`, `scripts/` within each skill).
- **Skill descriptions ≤ 1024 characters** — Codex's skill loader rejects
  longer descriptions. Keep them tight.
- **Self-contained skills** — a skill may name other skills as context but
  must never command their activation.
- **Do not add or remove symlinks here** — the symlink structure
  (`~/.claude/skills` → this repo, etc.) is managed by
  `stromy-org/scripts/agent-sync.py`. Run it from the control plane to
  reconcile.

## Editing a skill

1. Edit files directly in the skill directory.
2. Verify description length stays ≤ 1024 chars.
3. Commit via `/conventional-commit`.

## How machine-wide availability works

`stromy-org/global-skills/` is a git submodule. A whole-directory symlink
`~/.claude/skills → <stromy-org>/global-skills/` makes every skill here
available in every repo on the machine without per-repo copying. Codex uses
per-skill symlinks (`~/.codex/skills/conventional-commit → ...`).
