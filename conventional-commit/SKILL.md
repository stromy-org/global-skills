---
name: conventional-commit
description: Defines the repository-standard commit workflow using Conventional Commits and gitmoji. Triggers when creating commits in this repo. Includes a "clean source control" mode for guided git hygiene — triages changes into commit, gitignore, or revert.
---

# Conventional Commit

Create well-structured git commits following the Conventional Commits 1.0.0 specification with gitmoji support.

## 1. Commit Message Structure

A commit message MUST be composed of:

1. **Header** (single line, required)
2. **Body** (optional but recommended for non-trivial changes)
3. **Footer** (optional; required for breaking changes and issue closing references)

### 1.1 Header Grammar (strict)

Header MUST match:

```
<type>(<scope>): <gitmoji> <subject>
```

Where:
- `<type>` is REQUIRED and MUST be one of the allowed types (see §2)
- `<scope>` is OPTIONAL but recommended; describes the affected area or component
- `:` is REQUIRED after `<type>` or `<type>(<scope>)`
- `<gitmoji>` is REQUIRED and MUST appear **after the colon and a space** (see §3)
- `<subject>` is REQUIRED and MUST follow subject rules (see §4)

### 1.2 Scope Naming Rules

The scope should identify the **component, module, or area** affected by the change:

- Use the **parent directory or module name** (e.g., `skills`, `api`, `auth`, `db`)
- For skill changes: use `skills` as scope, mention the specific skill in the subject
- For nested components: use the most specific relevant scope (e.g., `api` not `src/api`)
- Keep scopes short and consistent across the project

**Scope examples:**
- `docs(skills): 📝 Add type selection rules to conventional-commit skill`
- `feat(api): ✨ Add user endpoint`
- `fix(auth): 🐛 Resolve token refresh issue`

#### Valid examples
- `feat(skills): ✨ Add FastMCP development skill packs`
- `fix(db): 🐛 Handle null schema version in migration runner`
- `docs: 📝 Update installation instructions`

#### Invalid examples (do not generate)
- `✨ feat: add ...` (emoji before type breaks tooling)
- `feat : add ...` (space before colon)
- `feat(scope) add ...` (missing colon)

**Format rules:**
- Type is lowercase (`feat`, `fix`, `docs`)
- Gitmoji appears after colon and space
- Subject starts with uppercase letter
- No period at end of subject
- Keep subject under 50 characters when possible

## 2. Commit Types with Gitmoji

| Type | Gitmoji | Code | When to use |
|------|---------|------|-------------|
| `feat` | ✨ | `:sparkles:` | New feature |
| `fix` | 🐛 | `:bug:` | Bug fix |
| `fix` | 🚑️ | `:ambulance:` | Critical hotfix |
| `docs` | 📝 | `:memo:` | Documentation only |
| `style` | 🎨 | `:art:` | Code formatting, structure |
| `refactor` | ♻️ | `:recycle:` | Code refactoring |
| `perf` | ⚡️ | `:zap:` | Performance improvement |
| `test` | ✅ | `:white_check_mark:` | Add or update tests |
| `build` | 📦️ | `:package:` | Build system, dependencies |
| `ci` | 👷 | `:construction_worker:` | CI configuration |
| `chore` | 🔧 | `:wrench:` | Configuration, tooling |
| `revert` | ⏪️ | `:rewind:` | Revert previous commit |

### 2.1 Type Selection Rules (deterministic)

When determining the commit type, follow this priority order:

1. If the change introduces a new user-facing capability → `feat`
2. Else if the change corrects faulty behavior → `fix`
3. Else if only docs changed → `docs`
4. Else if only tests changed → `test`
5. Else if performance improvement without feature/bug fix → `perf`
6. Else if internal restructure only → `refactor`
7. Else if only formatting/lint → `style`
8. Else if build/deps → `build`
9. Else if CI/pipeline → `ci`
10. Else → `chore`
11. If commit is explicitly undoing another commit → `revert`

> **Priority rule:** If multiple categories apply, choose the highest-priority user-facing intent: `fix`/`feat` > `perf` > `refactor` > `build/ci/chore` > `docs/test/style`.

### 2.2 Additional Gitmoji

| Gitmoji | Code | When to use |
|---------|------|-------------|
| 🔥 | `:fire:` | Remove code or files |
| 🚀 | `:rocket:` | Deploy stuff |
| 💄 | `:lipstick:` | UI and style updates |
| 🎉 | `:tada:` | Initial commit |
| 🔒️ | `:lock:` | Security fixes |
| ⬆️ | `:arrow_up:` | Upgrade dependencies |
| ⬇️ | `:arrow_down:` | Downgrade dependencies |
| ➕ | `:heavy_plus_sign:` | Add dependency |
| ➖ | `:heavy_minus_sign:` | Remove dependency |
| 🚚 | `:truck:` | Move or rename files |
| 💥 | `:boom:` | Breaking changes |
| 🩹 | `:adhesive_bandage:` | Simple fix for non-critical issue |
| ⚰️ | `:coffin:` | Remove dead code |
| ✏️ | `:pencil2:` | Fix typos |
| 🏗️ | `:building_construction:` | Architectural changes |

## 3. Breaking Changes

Indicate breaking changes with:
- Add an exclamation mark after the type (e.g., feat with exclamation, fix with exclamation)
- Or use the footer: BREAKING CHANGE: description

## 4. Subject Rules

### 4.1 Recommended Verbs

Start subjects with one of these imperative verbs:

`add`, `update`, `remove`, `fix`, `prevent`, `support`, `refactor`, `simplify`, `improve`, `optimize`, `document`, `rename`, `deprecate`

### 4.2 Subject Guidelines

- Use imperative mood ("Add" not "Added" or "Adds")
- Start with uppercase letter
- No period at end
- Keep under 50 characters when possible
- Be specific about what changed

## 5. Workflow

1. **Check branch**: Detect if on `main` or `master`. If so, apply the main-branch protection flow (see §5.2). If already on a feature branch, continue normally.
2. **Review changes**: Run `git status` and `git diff` to understand the full working tree.
3. **Choose commit strategy (single vs multiple)**:
   - Use **single commit** when all changes are one coherent logical unit with one dominant type/scope.
   - Use **multiple commits** when changes are independent units, mixed intent/types, or safer to review/revert separately.
   - Decide strategically; do not force multiple commits by default.
4. **Apply staging boundary mode**:
   - **Default mode (same implementation conversation):** stage only files changed for the work done in this session.
   - **Batch-review mode (explicit user request in a new conversation):** you may review all tracked changes, group/batch them, and commit selected groups after explicit user confirmation.
5. **Filter excluded files**: never include ephemeral artifacts/plans (see §7.1 exclusions).
6. **Execute commits directly** (no confirmation step — proceed immediately after deciding the commit plan):
7. **Selective staging**:
   - Stage explicit paths only (`git add <paths>` / `git add -u <paths>`).
   - Do **not** use blanket staging (`git add -A`, `git add .`, or `git commit -a`) unless the approved plan explicitly covers the entire tree.
   - Verify staged content before each commit (`git diff --cached --name-only`).
8. **Offer PR creation**: Ask if user wants to create a pull request.

### 5.1 Commit Splitting Strategy

Decide autonomously how many commits to create based on these rules:

- **Single commit** when all changes are part of the same logical unit (e.g., one feature, one bug fix, one refactor)
- **Multiple commits** when changes span different concerns — split them into separate, atomic commits

**Split signals** (create separate commits):
- Different commit types apply (e.g., a `feat` and a `docs` change)
- Changes touch different file categories (source code, tests, docs, config) that aren't tightly coupled
- Changes touch unrelated modules or scopes
- A refactor was done alongside a new feature
- Test additions are unrelated to the main change
- Config/build changes are independent of feature work

**Keep together signals** (single commit):
- All changes serve the same purpose with one dominant type
- Tests are directly for the feature/fix being committed (same logical unit)
- Style/formatting changes are part of the same refactor

When splitting, stage files selectively with `git add <file>` (not `git add .`) and commit each logical unit in sequence. Always commit in dependency order (e.g., refactor before feature that depends on it).

### 5.2 Main Branch Protection

When the current branch is `main` or `master`, do NOT ask — handle it automatically:

1. Review **all** changes first and decide the commit strategy (single or multiple commits per §5.1)
2. Derive **one** branch name from the overall theme of the changes: `<type>/<kebab-case-theme>` (e.g., `feat/brand-integration-improvements`, `fix/auth-token-handling`). When multiple commit types are involved, use the highest-priority type (§2.1) for the branch prefix.
3. Create and switch: `git checkout -b <branch>`
4. Make **all** commits on this branch — stage and commit each logical unit in sequence (§5.1, §9)
5. Switch back: `git checkout main` (or `master`)
6. **No-fast-forward merge**: `git merge --no-ff <branch>` — this creates a single merge commit preserving the branch topology in the git graph
7. Delete the branch: `git branch -d <branch>`

**Key rule**: One branch per invocation, one merge. Multiple commits go on the **same** branch — never create a separate branch per commit. This preserves branch-per-change discipline with visible merge topology. No user confirmation needed — just do it and report what happened.

## 6. Examples

**Example 1 - New feature with scope:**
```
feat(auth): ✨ Add user authentication system
```

**Example 2 - Bug fix with scope:**
```
fix(api): 🐛 Resolve null pointer in user service
```

**Example 3 - Documentation (no scope):**
```
docs: 📝 Update API documentation for v2 endpoints
```

**Example 4 - Refactoring with scope:**
```
refactor(validation): ♻️ Simplify token validation logic
```

**Example 5 - Breaking change:**
```
feat(config): 💥 Change configuration file format

BREAKING CHANGE: config.json replaced with config.yaml
```
Note: For breaking changes, add exclamation mark after type or use BREAKING CHANGE footer.

**Example 6 - With body:**
```
feat(auth): ✨ Add OAuth2 login support

Implement OAuth2 flow with Google and GitHub providers.
Includes token refresh and session management.
```

## 7. Rules

- One logical change per commit
- Subject in imperative mood using recommended verbs ("Add" not "Added")
- Split unrelated changes into separate commits
- Gitmoji should match the commit type
- Body explains "what" and "why", not "how"
- Commit strategy must be intentional: single commit is valid when strategically cleaner
- Default to session-scoped commits; do not auto-include unrelated tracked changes
- In batch-review mode, include only user-approved groups after showing grouped file lists

### 7.1 Hard Exclusions (never auto-commit)

The following must never be committed by this skill:

- Plan/spec scratch docs: `PLAN_*.md`, `RUN_*.md`
- Temporary/editor/system artifacts: `.DS_Store`, `*.log`, `*.tmp`, `*.bak`
- Runtime/generated artifacts under workspace outputs (for example run outputs, caches, ad-hoc export files)

If unsure whether a file is an artifact, treat it as excluded and ask before staging.

## 8. Clean Source Control Mode

An assisted mode for users who need help understanding what git is tracking and what to do about it. Activate with `/conventional-commit clean` or when the user asks to "clean up source control", "help with git status", or "tidy my repo".

### 8.1 Purpose

Many users end up with a messy working tree — untracked files that should be ignored, accidental changes that should be reverted, and real work that should be committed. This mode triages **every** change and guides the user through resolving it.

### 8.2 Workflow

1. **Full inventory**: Run `git status` and present a clear, grouped summary of:
   - **Staged changes** (files already in the index)
   - **Unstaged modifications** (tracked files with changes)
   - **Untracked files** (new files git doesn't know about)

2. **Triage each item**: For every file or group of related files, decide and recommend one of three actions:

   | Action | When | Example |
   |--------|------|---------|
   | **Commit** | The change is intentional work that should be preserved | Source code, config, docs |
   | **Gitignore** | The file should never be tracked — it's generated, personal, or environment-specific | `.DS_Store`, `*.pyc`, `.env`, IDE configs, build artifacts, `node_modules/` |
   | **Revert** | The change is accidental or unwanted — restore to the last committed state | Unintended edits, debug prints left behind, scratch changes |

3. **Present the triage plan**: Show a clear table grouping files by recommended action:

   ```
   ## Triage Plan

   ### ✅ Commit (3 files)
   - src/auth.py — New authentication module
   - tests/test_auth.py — Tests for auth module
   - CLAUDE.md — Updated project instructions

   ### 🙈 Gitignore (2 files)
   - .DS_Store — macOS system file
   - __pycache__/ — Python bytecode cache

   ### ⏪ Revert (1 file)
   - src/main.py — Only change is a debug print statement
   ```

4. **Ask for confirmation**: Present the plan and ask the user to confirm or adjust. Unlike the standard commit workflow, this mode **always asks before acting** — the user is learning, so transparency matters.

5. **Execute approved actions** in this order:
   1. **Gitignore first**: Append patterns to `.gitignore`, then `git rm --cached` any already-tracked files that match new ignore patterns
   2. **Revert second**: `git checkout -- <file>` for unstaged changes, `git reset HEAD <file>` then `git checkout -- <file>` for staged changes the user wants to discard
   3. **Commit last**: Stage and commit the approved files using the standard conventional-commit format (§1–§7)

6. **Explain as you go**: Briefly explain each git command before running it, so the user builds understanding. Keep explanations short — one sentence per command, not a tutorial.

### 8.3 Triage Decision Tree

```
For each changed/untracked file:
│
├─ Is it a generated artifact, cache, or OS file?
│  (.DS_Store, __pycache__, *.pyc, node_modules/, .env, *.log, build/, dist/)
│  → GITIGNORE
│
├─ Is it an IDE/editor config?
│  (.vscode/, .idea/, *.swp, *.swo)
│  → GITIGNORE (unless project shares editor config intentionally)
│
├─ Is it a secret or credential?
│  (.env, *.key, *.pem, credentials.json, service-account.json)
│  → GITIGNORE + WARN user if it was ever committed
│
├─ Is the diff trivial/accidental?
│  (only whitespace, a stray print/console.log, no semantic change)
│  → REVERT
│
├─ Is it intentional work?
│  → COMMIT (group with related files into logical commits per §5.1)
│
└─ Unsure?
   → ASK the user — show the diff and let them decide
```

### 8.4 Rules

- **Always confirm** before reverting or ignoring — never silently discard work
- **Group related files** when presenting the triage plan (e.g., "these 3 Python cache files" not listing each one)
- **Check existing `.gitignore`** before suggesting additions — don't duplicate patterns
- **Warn about secrets**: If a file looks like a credential and is tracked, flag it prominently
- **No judgment**: Use neutral language — the user is learning, not being evaluated
- After cleanup, show the final `git status` so the user can see the clean state

## 9. Commit Execution

Use HEREDOC for proper message formatting:

```bash
git commit -m "$(cat <<'EOF'
type(scope): <gitmoji> Subject

Optional body explaining the change.

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```
