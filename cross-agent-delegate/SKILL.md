---
name: cross-agent-delegate
description: When and how to delegate work between AI coding agents (Claude Code, Codex CLI, Gemini CLI) via plugins or shell-out — including the reverse direction (Codex/Gemini calling Claude). Covers write-capable delegation, sandbox/trust setup for all three agents, and the supervisor review pattern that replaces per-tool approval gates. Triggers on "use codex/gemini for this", huge-context tasks, second-opinion reviews, or any cross-agent delegation request.
---

# Cross-Agent Delegation

Three coding agents are available on this machine: **Claude Code**, **Codex CLI** (`codex`), and **Gemini CLI** (`gemini`). Each can invoke the others, but cross-agent calls are **opt-in**, not a default reflex.

This skill exists so that delegation is a deliberate choice with a clear reason — not a token-saving habit that fragments context, hides bugs across three transcripts, and quietly bypasses the host agent's permission model.

## 1. The default is: do not delegate

The current agent should finish the task in-process unless **one of the trigger conditions in §2 holds**. "It might save credits" is not a trigger condition — see §6 for why.

If unsure, do the work yourself. Cross-agent delegation is the exception.

## 2. When delegation is appropriate

Delegate only when one of these is true:

| Trigger | Why | Reach for |
|---------|-----|-----------|
| User explicitly asked ("use codex for this", "ask gemini", "/codex:review") | User intent overrides default | Whichever they named |
| Input is huge (50k+ lines, repo-wide grep/summary) AND the output is small (a summary, a list, an answer) | Context offload + cost-shift to Google quota | Gemini (1M context) |
| Task involves visual assets (screenshots, diagrams, images, audio/video) OR needs live web data during analysis | Gemini 3 reasons natively over all modalities in one pass; has built-in Google Search grounding that Claude and Codex lack | Gemini |
| Shell/terminal-centric task (scripting, CLI tooling, system operations) where reasoning quality isn't the bottleneck | Codex scores 77.3% vs 65.4% on Terminal-Bench 2.0 and uses 3–4× fewer tokens per task | Codex |
| Bulk task that doesn't need Opus-grade reasoning (boilerplate scan, doc generation, repetitive refactor across many files) AND Claude isn't a quality requirement | Cost-shift to Google/OpenAI quota | Gemini for breadth, Codex for adversarial precision |
| Independent second opinion on a finished diff or plan, before merge | Different model behavior is the whole point | Codex (`/codex:review` or `/codex:adversarial-review`) |
| Background long-running task while user keeps working | `--background` mode exists for this | Codex `/codex:rescue --background` |

If none of these hold, **do not delegate.**

### On the cost-shift case (important nuance)

Cross-agent delegation **does** shift compute off Claude when the output is small relative to the input. Gemini reading 200k lines and returning a 2KB summary genuinely offloads work to your Google quota — you pay Claude only for the 2KB return.

But the saving evaporates when:
- The delegated agent returns large output (you pay Claude for all of it on re-entry).
- You're well below your Claude Max cap (you're substituting paid-by-Google tokens for tokens already covered by your flat Anthropic plan).
- The task needs reasoning quality you'd lose by switching off Opus.

**Decide on the actual constraint:** if you're hitting Claude rate limits or context bloat, delegate. If you're just running normally, don't.

## 3. Hard rules (apply in all cases)

1. **Writes require explicit authorization + supervisor review.** Read/analyze is the safe default when the delegation is open-ended. When the user explicitly authorizes writes ("have Codex implement this", "let Codex make the changes"), writes are allowed — but the **supervisor contract (§8) applies**: Claude reviews the output before accepting it, never blindly.
2. **No spontaneous chains.** The delegated agent must not, in turn, delegate further. One hop max unless the user explicitly chains them.
3. **No "save tokens" delegation.** Forks (`Agent` without `subagent_type` in Claude) already share the prompt cache and offload tool noise — they are nearly free. Cross-agent calls cost the *other* provider's tokens and lose cache. Do not reach for Codex/Gemini to save Claude credits; use a fork instead.
4. **Surface the cost to the user.** Before the first cross-agent call in a session, tell the user one short sentence: "Delegating to Codex for an independent review — this bills against your OpenAI key." After the first, you can stay quiet.
5. **Commits and pushes always go through the host.** The delegated agent stages and writes files; the host agent (Claude) runs the commit via the `conventional-commit` skill and pushes under its own permission rules.

## 4. Picking the model and reasoning effort

Each CLI exposes a different lever. **Match the lever to the task — don't pay heavy-reasoning prices for mechanical work, and don't run light models on hard reasoning.**

The default rule across all three: **tighten the prompt before escalating effort.** Better contracts beat higher reasoning. Only raise effort when the task genuinely requires deeper search, planning, or adversarial scrutiny.

> **Cost reference:** `references/agent-model-selection.md` (in this skill's directory) has live-fetched pricing, quality/price ratios, and a sweet-spot table per task shape. **On a Claude Max flat-rate plan, in-process Claude work has zero marginal cost — every cross-agent call adds cost against your OpenAI or Google quota.** The tables below show model character and task fit; see the reference for exact per-MTok figures.

### Claude Code (host)

| Model | When |
|-------|------|
| **Opus 4.7** (default for stromy-org) | Architectural decisions, deep refactors, multi-file reasoning, ambiguous specs, supervisor review of risky diffs |
| **Sonnet 4.6** | Most day-to-day work — feature implementation, well-scoped fixes, doc edits with judgment |
| **Haiku 4.5** | Mechanical edits, tight loops, file shuffling, one-shot lookups, anything where the answer is obvious once you read the file |

Switch with `/model`. `/fast` toggles Opus 4.6 fast-path on Opus 4.6 sessions.
For effort tier on a single task, the `/effort` skill exposes low/medium/high — leave at default unless a task is clearly a one-liner (lower) or a thorny cross-cutting design call (higher).

### Codex CLI (`gpt-5.5` family)

Model and effort are **separate axes**. Defaults in `~/.codex/config.toml` are `model = "gpt-5.5"`, `model_reasoning_effort = "high"`.

| Flag | Values | Notes |
|------|--------|-------|
| `--model` | `gpt-5.5` (default), `spark` (→ `gpt-5.3-codex-spark`), or any explicit Codex model id | `spark` is the lightweight/fast variant — use for mechanical fixes |
| `--effort` | `none` · `minimal` · `low` · `medium` · `high` · `xhigh` | Only override when justified |

**Effort selection — when delegating via `/codex:rescue` or `/codex:review`:**

| Effort | Use for |
|--------|---------|
| `minimal` / `low` | Boilerplate scans, mechanical refactors with explicit instructions, "rename X to Y everywhere" |
| `medium` | Default for most rescues, normal review passes, well-scoped bug fixes |
| `high` | Diagnosis without a clear hypothesis, cross-file refactors, adversarial review, "why is this flaky" |
| `xhigh` | Last resort. Genuine architecture calls, deeply tangled bugs, when `high` ran twice and didn't converge |

**Model selection:**
- Use `--model spark` when the task is mechanical and you want speed (e.g., "apply this exact rename across the repo", "regenerate snapshots").
- Leave `--model` unset (default `gpt-5.5`) for everything else — including review.

**Per the embedded `gpt-5-4-prompting` guidance: "Do not raise reasoning or complexity first. Tighten the prompt and verification rules before escalating."** If a `medium`-effort run fails, fix the prompt before bumping to `high`.

### Gemini CLI (`gemini-3-pro` / `gemini-2.5-pro` / `gemini-2.5-flash`)

Gemini does **not** expose a reasoning-effort flag — thinking budget is allocated automatically. **Model choice is the effort lever.**

| Model | When |
|-------|------|
| **`gemini-3-pro`** | Default. Multimodal reasoning, Google Search grounding, deepest reasoning. Use for architecture passes, security audits, anything with images/diagrams, anything needing fresh web facts |
| **`gemini-2.5-pro`** | When you specifically want 2.5-Pro's behavior (more conservative, well-known output style) — rarely the right pick now that 3-pro is GA |
| **`gemini-2.5-flash`** | Bulk doc generation, "summarize these 200 files", high-throughput repo scans where reasoning depth doesn't matter |

Pass via the bridge: `/cc-gemini-plugin:gemini --model gemini-2.5-flash --dirs src "list every public function and its docstring"`.

If unset, the bridge uses Gemini CLI's configured default — fine for most analysis tasks.

### Cross-CLI selection cheatsheet

| Task shape | Pick |
|------------|------|
| Architecture / risky decision | Claude Opus 4.7 (host, no delegation) |
| Independent review of a finished diff | `/codex:review` at default effort |
| Adversarial review, finding subtle bugs | `/codex:adversarial-review`, escalate to `--effort high` only if first pass is shallow |
| Whole-repo summary or grep over 50k+ lines | Gemini 3-pro via bridge |
| Bulk mechanical edits across many files | Codex `--model spark --effort low` OR Gemini 2.5-flash |
| Visual / diagram / screenshot reasoning | Gemini 3-pro (only one that's natively multimodal here) |
| Fresh web facts mid-analysis | Gemini 3-pro (built-in Search grounding) |
| Long-running background fix | `/codex:rescue --background` at default effort |

## 5. How to invoke

### From Claude Code

**Free-form delegation (the primary entry points):**
- **Codex with arbitrary task:** `/codex:rescue "<what codex should do>"` — accepts any instruction, including "go invoke the /<repo-skill> skill in this repo on the current branch and report back"
- **Gemini with arbitrary task:** `/cc-gemini-plugin:gemini --dirs <paths> --files <glob> "<task>"` — accepts any task; same free-form contract as `/codex:rescue`

Both subagents inherit access to the current repo's `.claude/skills/` and `.agents/skills/` directories, so they can run repo-local skills (e.g. `/review`, a custom builder skill) on your behalf. Use this when the *workflow* is yours but you want the *execution* on a different model.

**Pre-shaped Codex commands (use when they fit; otherwise use `/codex:rescue`):**
- `/codex:review` — review the current diff
- `/codex:adversarial-review` — hostile review
- `/codex:rescue --background "<task>"` — long-running task in background
- `/codex:status` / `/codex:result` / `/codex:cancel` — manage background tasks
- `/codex:setup` — one-time auth check

**Gemini subagent form:** invoke the `gemini-agent` subagent via the `Agent` tool when you want Claude to delegate as part of a larger plan rather than a one-shot user-issued command. Gemini CLI can itself orchestrate parallel subagents natively — if the delegated task decomposes into independent subtasks, Gemini can fan those out internally; you don't need to manage the parallelism from Claude's side.

### From Codex CLI or Gemini CLI (reverse direction)

There is no installed plugin for the reverse direction. Shell out: `claude -p "<prompt>"`.

**Scope the call tightly.** Give Claude a bounded task with explicit file targets and a clear return format — not "fix the codebase." Claude should return a diff summary or a list of changed files in its response so the calling agent can review it.

**Review gate applies in reverse too.** The supervisor contract (§8) is symmetrical — the calling agent (Codex or Gemini) must review Claude's output before accepting it, not pipe it straight to disk:

1. Read Claude's response: what files did it change, what was the logic?
2. Spot-check the changed files against the stated scope.
3. Accept / iterate (max 2 passes) / rescue — same thresholds as §8.

**Commits and pushes always go through the top-level agent.** Whichever agent started the session owns the commit. Claude should not commit or push when invoked as a sub-agent via `-p`; it writes files and reports back, the calling agent commits.

This is a write-class action — always surface it to the user before the first call in a session: "Using `claude -p` to delegate X to Claude — this bills against your Anthropic key."

## 6. Why "save Claude tokens" is the wrong reason

- The delegated agent's output still lands in the host agent's context as a tool result. The host still bills for that input.
- The host's prompt cache is preserved across forks but **not** across cross-agent calls — every Codex/Gemini call starts cold.
- Codex and Gemini calls bill against their own API quotas. On a flat-rate Claude Max plan, this typically *increases* total spend.
- If the goal is "keep large intermediate output out of my context," use a Claude **fork** (`Agent` tool, no `subagent_type`). That is the cheap, cache-warm option.

Use cross-agent delegation when you want a different *model's behavior*, not when you want a cheaper token.

## 7. Sandbox and trust setup

### Codex

For Codex to make file edits non-interactively (no human at a terminal to approve), the repo must be in the global trusted list (`~/.codex/config.toml`). Every org repo is listed there with `trust_level = "trusted"`.

Project configs use `sandbox_mode = "workspace-write"` + `approval_policy = "on-request"` for interactive sessions. In trusted repos Codex auto-approves workspace file writes and shell commands scoped to the workspace — network calls and external filesystem access are still sandboxed.

If Codex is blocked mid-task: check that the repo path is in `~/.codex/config.toml`. Missing entries are the most common cause of unexpected "sandbox permissions" errors in non-interactive delegation.

### Gemini

The Gemini equivalent of Codex's trusted-list is `defaultApprovalMode: "auto"` in the project's `.gemini/settings.json`. Every org repo's settings file uses `"auto"` so Gemini CLI can operate non-interactively when delegated.

`"tools": { "sandbox": true }` remains enabled — this restricts network and external filesystem access while still allowing workspace writes. The review gate for Gemini output is the same supervisor contract (§8) as Codex.

If Gemini is blocked mid-task: verify the repo's `.gemini/settings.json` has `"defaultApprovalMode": "auto"`, not `"default"`.

## 8. Supervisor review pattern

When Codex (or Gemini) is delegated **write work**, the host agent (Claude) must review before accepting — never blindly. The review is brief, not exhaustive.

**After the delegated agent returns:**

1. **Read the diff** — `git diff HEAD` or `git diff --name-only HEAD` to see what changed.
2. **Spot-check** — read changed files in key areas: entry points, schema definitions, anything the task description touched. One pass, not a full audit.
3. **Assess scope creep** — did the agent touch files outside the stated scope? Flag and revert any out-of-scope changes.
4. **Decide:**

| Outcome | Action |
|---------|--------|
| Work is complete and correct | Accept: proceed with commit, continue task |
| Work is incomplete or has specific fixable gaps | **Iterate**: send Codex back with precise feedback — name the exact files/functions that are wrong and what the fix should be |
| Work is wrong in a way that needs architectural judgment | **Rescue**: Claude picks it up directly; do not keep cycling Codex on something that needs reasoning the task didn't set up for |
| Codex wrote nothing / failed silently | Check sandbox trust setup (§7); re-delegate with a simpler, more targeted prompt |

**When iterating**, be specific: "Files `src/agent.py` lines 45–62 still use the old `LLMChain` API. Replace with `RunnableSequence`. Everything else looks good." Vague re-delegation loops waste quota.

**Maximum iterations: 2.** If Codex hasn't converged after two passes, rescue it — the task likely needs a different decomposition.

## 9. Failure modes to watch

- **Stale auth** — Codex/Gemini token expired silently; tool returns garbage. If a delegated call returns nonsense, suspect auth before suspecting the model.
- **Sandbox mismatch** — the delegated agent runs under its own approval policy, not the host's. A hook you rely on in Claude does not fire inside a Codex sub-call.
- **Transcript fragmentation** — when something goes wrong, you now have three logs to correlate. Keep delegated calls small and self-contained so the failure surface stays narrow.
- **Recursion / loop billing** — never let agent A call agent B which calls agent A. Enforce one hop max (§3.2).

## 10. Quick decision tree

```
Did the user explicitly ask for codex/gemini?
├── Yes → delegate; writes allowed; apply supervisor contract (§8)
└── No  → Is the input huge AND output small?
         ├── Yes → consider Gemini (one-shot, summary-only)
         └── No  → Do you want an independent second opinion on finished work?
                  ├── Yes → consider /codex:review
                  └── No  → Do it yourself. Use a fork if you want context isolation.
```

## 11. Maintenance

**§4 (model + reasoning effort) and `references/agent-model-selection.md` decay as model lineups and prices evolve.** Re-check whenever any of the following happens:

- A new Claude / GPT / Gemini model GAs (e.g., Opus 4.8, GPT-5.6, Gemini 4)
- An old model is retired or repriced
- A CLI adds, removes, or renames a flag (`codex --help`, `gemini --help`, `/codex:rescue`'s argument-hint)
- A CLI changes its default model or default reasoning effort
- `references/agent-model-selection.md` last-updated date exceeds 90 days (flagged by `instruction-audit` drift scan)

**Update procedure — always fetch live, never use training memory for pricing:**

```
1. Fetch live pricing pages (URLs in references/agent-model-selection.md § Maintenance):
     https://www.anthropic.com/pricing
     https://openai.com/api/pricing/
     https://ai.google.dev/pricing

2. Verify current CLI flag→model mappings:
     codex --help  (or: codex exec --help)
     gemini --help

3. Check benchmark leaderboards if quality ordering has plausibly shifted:
     https://www.swebench.com/   (SWE-bench Verified)
     Search GitHub/arXiv for "Terminal-Bench" for latest run

4. Update references/agent-model-selection.md:
     - Pricing tables in §2
     - Quality/price ratios in §3 if rankings shifted
     - "Last updated" date at the top

5. Update §4 of this file:
     - Per-CLI model tables (names, cost tiers, effort flags)
     - Cross-CLI cheatsheet if sweet spots changed
     - spark/flash alias footnotes if CLI flag names changed

6. Update defaults referenced in ~/.codex/config.toml if default model changed.
```

When in doubt about current CLI surface, re-run `codex --help`, `codex exec --help`, `gemini --help`, and skim the latest plugin command files at `~/.claude/plugins/cache/{openai-codex,cc-gemini-plugin}/**/commands/*.md` — those are authoritative, not this doc.
