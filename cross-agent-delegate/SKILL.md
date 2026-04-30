---
name: cross-agent-delegate
description: When and how to delegate work from one AI coding agent (Claude Code, Codex CLI, Gemini CLI) to another via installed plugins or shell-out, instead of doing it in-process. Triggers on huge-context scans, second-opinion reviews, or explicit "use codex/gemini for this" requests. Hard rule — never delegate spontaneously to save tokens; only delegate when the task profile clearly fits or when the user asked for it.
---

# Cross-Agent Delegation

Three coding agents are available on this machine: **Claude Code**, **Codex CLI** (`codex`), and **Gemini CLI** (`gemini`). Each can invoke the others, but cross-agent calls are **opt-in**, not a default reflex.

This skill exists so that delegation is a deliberate choice with a clear reason — not a token-saving habit that fragments context, hides bugs across three transcripts, and quietly bypasses the host agent's permission model.

## 1. The default is: do not delegate

The current agent should finish the task in-process unless **one of the trigger conditions in §2 holds**. "It might save credits" is not a trigger condition — see §5 for why.

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

1. **Read/analyze only by default.** A delegated call may read files and return a summary, review, or answer. It must not write files, run migrations, push commits, or modify shared state without the user explicitly approving that specific delegated write.
2. **No spontaneous chains.** The delegated agent must not, in turn, delegate further. One hop max unless the user explicitly chains them.
3. **No "save tokens" delegation.** Forks (`Agent` without `subagent_type` in Claude) already share the prompt cache and offload tool noise — they are nearly free. Cross-agent calls cost the *other* provider's tokens and lose cache. Do not reach for Codex/Gemini to save Claude credits; use a fork instead.
4. **Surface the cost to the user.** Before the first cross-agent call in a session, tell the user one short sentence: "Delegating to Codex for an independent review — this bills against your OpenAI key." After the first, you can stay quiet.
5. **Never let cross-agent output bypass approval.** If the delegated agent suggests a change, the host agent applies it under the host's normal permission/hook rules. Do not pipe the delegated agent's diff straight to disk.

## 4. How to invoke

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

### From Codex CLI or Gemini CLI
There is no installed plugin for the reverse direction. If you genuinely need Claude from inside Codex/Gemini, shell out: `claude -p "<prompt>"`. Treat this as a write-class action — confirm with the user first, every time.

## 5. Why "save Claude tokens" is the wrong reason

- The delegated agent's output still lands in the host agent's context as a tool result. The host still bills for that input.
- The host's prompt cache is preserved across forks but **not** across cross-agent calls — every Codex/Gemini call starts cold.
- Codex and Gemini calls bill against their own API quotas. On a flat-rate Claude Max plan, this typically *increases* total spend.
- If the goal is "keep large intermediate output out of my context," use a Claude **fork** (`Agent` tool, no `subagent_type`). That is the cheap, cache-warm option.

Use cross-agent delegation when you want a different *model's behavior*, not when you want a cheaper token.

## 6. Failure modes to watch

- **Stale auth** — Codex/Gemini token expired silently; tool returns garbage. If a delegated call returns nonsense, suspect auth before suspecting the model.
- **Sandbox mismatch** — the delegated agent runs under its own approval policy, not the host's. A hook you rely on in Claude does not fire inside a Codex sub-call.
- **Transcript fragmentation** — when something goes wrong, you now have three logs to correlate. Keep delegated calls small and self-contained so the failure surface stays narrow.
- **Recursion / loop billing** — never let agent A call agent B which calls agent A. Enforce one hop max (§3.2).

## 7. Quick decision tree

```
Did the user explicitly ask for codex/gemini?
├── Yes → use it, follow §3 hard rules
└── No  → Is the input huge AND output small?
         ├── Yes → consider Gemini (one-shot, summary-only)
         └── No  → Do you want an independent second opinion on finished work?
                  ├── Yes → consider /codex:review
                  └── No  → Do it yourself. Use a fork if you want context isolation.
```
