# Global Skills

Machine-wide Claude Code and Codex CLI skills — a single source of truth for skills that should be available in every repository on this machine.

## Skills

| Skill | Description | Upstream |
|-------|-------------|----------|
| `conventional-commit` | Conventional Commits with gitmoji | stromy-org (org standard) |
| `skill-creator` | Create, improve, and evaluate skills | [anthropics/skills](https://github.com/anthropics/skills) |

## How It Works

This repo is added as a **submodule** in `stromy-org` and **symlinked** to the user-level skill directories for both Claude Code and Codex CLI. One set of files, discovered by both tools.

```
global-skills/                          # Git repo (submodule of stromy-org)
├── conventional-commit/SKILL.md
├── skill-creator/
│   ├── SKILL.md
│   ├── agents/
│   ├── eval-viewer/
│   ├── references/
│   ├── scripts/
│   └── LICENSE.txt
└── README.md                           # This file
```

### Symlink Layout

```
~/.claude/skills    → <stromy-org>/global-skills/     # Whole directory symlink
~/.codex/skills/conventional-commit → <stromy-org>/global-skills/conventional-commit  # Per-skill symlink
```

**Why per-skill for Codex?** Codex manages its own built-in skills in `~/.codex/skills/.system/` (including its own `skill-creator`). Replacing the whole directory would wipe those out. Claude Code has no such constraint.

### Skill Discovery Hierarchy

| Level | Claude Code | Codex CLI |
|-------|------------|-----------|
| **User-level (global)** | `~/.claude/skills/` | `~/.codex/skills/` |
| **Project-level** | `.claude/skills/` | `.agents/skills/` |
| **Plugin-level** | `.claude-plugin/` | N/A |

User-level skills are available in every repository. Project-level skills are scoped to a single repo. When both exist, project-level takes precedence.

## Setup on a New Machine

After cloning stromy-org with submodules:

```bash
# 1. Init submodules
cd ~/Repositories/stromy-org
git submodule update --init --recursive

# 2. Symlink for Claude Code (whole directory)
rm -rf ~/.claude/skills 2>/dev/null
ln -s "$(pwd)/global-skills" ~/.claude/skills

# 3. Symlink for Codex CLI (per-skill, preserves .system/)
ln -s "$(pwd)/global-skills/conventional-commit" ~/.codex/skills/conventional-commit
# Don't symlink skill-creator — Codex has its own in .system/
```

## Updating Skills

### conventional-commit

Maintained in this repo. Edit `conventional-commit/SKILL.md` directly, commit, and push.

### skill-creator

Upstream source: `stromy-org/upstream/anthropic-skills/skills/skill-creator`. When Anthropic updates their skill-creator:

1. Update the upstream submodule: `git submodule update --remote upstream/anthropic-skills`
2. Compare changes: `diff -r upstream/anthropic-skills/skills/skill-creator global-skills/skill-creator`
3. Cherry-pick relevant changes into `global-skills/skill-creator/`
4. Commit in global-skills, then update the submodule pointer in stromy-org

## Impact on Bootstrap

Since `conventional-commit` and `skill-creator` are now globally available, the `repo-bootstrap` skill **no longer copies them** into satellite repos. Bootstrap generates 2 project-level skills (not 4):

- `quality-check` — repo-specific quality suite
- `instruction-audit` — repo-specific instruction maintenance

Satellite repos inherit `conventional-commit` and `skill-creator` automatically from the global skills directory.
