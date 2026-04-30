# GEMINI.md

Gemini CLI loads this file alongside AGENTS.md (see `.gemini/settings.json` →
`context.fileName`). AGENTS.md holds the canonical, self-contained instructions
for any AI agent working in this repo. Read it first.

This file is for **Gemini-specific addenda only** — config quirks, tool
availability differences, or workflow notes that do not apply to Claude Code or
Codex CLI. Keep it short. If a note applies to all agents, put it in AGENTS.md.

## Discovery paths Gemini uses in this repo

- Repo skills: `.agents/skills/` (Gemini reads it as a built-in alias — `.gemini/skills` is intentionally NOT created to avoid duplicate-skill warnings)
- MCP servers: declared in `.gemini/settings.json` `mcpServers`
  (generated from `.agents/mcp.json` — never hand-edit)
- Custom slash commands: `.gemini/commands/` (TOML files; subdirs become namespaces)
