# Agent Model Selection: Cost & Quality Reference

> **Last updated:** 2026-05-02  
> **⚠️ Pricing data must always be fetched live — never rely on training memory.**  
> See [Maintenance](#maintenance) for the exact procedure.

This document is the authoritative cost reference for cross-agent delegation decisions in stromy-org. It is linked from `global-skills/cross-agent-delegate/SKILL.md` §4 and checked by the `instruction-audit` drift scan.

---

## 1. Current Pricing

Prices fetched live on **2026-05-02** from the official pricing pages listed in [Maintenance](#maintenance).

### Anthropic — Claude models

| Model | Input $/MTok | Output $/MTok | Cache write $/MTok | Cache read $/MTok | Cost tier |
|-------|-------------|--------------|-------------------|-------------------|-----------|
| **Claude Haiku 4.5** | $1.00 | $5.00 | $1.25 | $0.10 | Cheap |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | $3.75 | $0.30 | Mid |
| **Claude Opus 4.7** | $5.00 | $25.00 | $6.25 | $0.50 | Premium |

Notes: Batch API gives 50% off. US-only inference adds ×1.1. Prompt cache TTL is 5 minutes.  
*On Claude Max, these are all flat-rate — the table matters only for API-key billing or rate-limit planning.*

### OpenAI — Codex CLI models

| Model | Input $/MTok | Cached input $/MTok | Output $/MTok | Cost tier | Codex CLI flag |
|-------|-------------|--------------------|--------------|-----------|-|
| **GPT-5.4 nano** | $0.20 | $0.02 | $1.25 | Ultra-cheap | verify with `codex --help` |
| **GPT-5.4 mini** | $0.75 | $0.075 | $4.50 | Cheap | verify with `codex --help` |
| **GPT-5.3-Codex** | $1.75 | $0.175 | $14.00 | Mid | verify with `codex --help` |
| **GPT-5.4** | $2.50 | $0.25 | $15.00 | Mid | verify with `codex --help` |
| **GPT-5.5** | $5.00 | $0.50 | $30.00 | Premium | default (`gpt-5.5`) |
| **GPT-5.5 Pro** | $30.00 | — | $180.00 | Very premium | not recommended for delegation |

Notes: Batch API gives 50% off. Regional processing adds 10%.  
**Important:** The Codex CLI uses model alias flags (`--model spark`, etc.). Alias→model-name mapping changes with CLI versions — always run `codex --help` to confirm current mappings before relying on a specific flag.

### Google — Gemini CLI models

| Model | Input $/MTok | Output $/MTok | Context window | Cost tier | Gemini CLI name |
|-------|-------------|--------------|----------------|-----------|----------------|
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | — | Ultra-cheap | verify with `gemini --help` |
| **Gemini 2.5 Flash** | $0.30 | $2.50 | — | Cheap | `gemini-2.5-flash` |
| **Gemini 3 Flash Preview** | $0.50 | $3.00 | — | Cheap | verify with `gemini --help` |
| **Gemini 2.5 Pro** | $1.25 (≤200K) / $2.50 (>200K) | $10.00 (≤200K) / $15.00 (>200K) | — | Mid | `gemini-2.5-pro` |
| **Gemini 3.1 Pro Preview** | $2.00 (≤200K) / $4.00 (>200K) | $12.00 (≤200K) / $18.00 (>200K) | — | Mid-premium | verify with `gemini --help` |

Notes: Batch API gives 50% off. Google Search tool costs $14/1,000 queries after free tier. Context caching available on Pro models.  
**Important:** Preview model names change frequently. `gemini --help` and the Gemini CLI changelog are authoritative — do not hardcode preview model names in automation.

---

## 3. Quality/Price Ratio

### Quality ordering for coding tasks (approximate, 2026-05-02)

Based on public benchmarks (SWE-bench Verified, Terminal-Bench 2.0) and the stromy-org operational record:

| Rank | Model | Coding quality | Cost tier | Best value for |
|------|-------|----------------|-----------|----------------|
| 1 | Claude Opus 4.7 | ★★★★★ | Premium | Architectural decisions, ambiguous specs, supervisor review |
| 1 | GPT-5.5 | ★★★★★ | Premium | Same as Opus — pick whichever model fits the task character |
| 2 | Claude Sonnet 4.6 | ★★★★ | Mid | Feature implementation, well-scoped fixes |
| 2 | GPT-5.4 | ★★★★ | Mid | Balanced coding tasks via Codex |
| 2 | Gemini 2.5 Pro | ★★★★ | Mid | Large-context passes, multi-repo summaries |
| 3 | Claude Haiku 4.5 | ★★★ | Cheap | Mechanical edits, tight loops, one-shot lookups |
| 3 | GPT-5.4 mini | ★★★ | Cheap | Mechanical Codex tasks (rename, regenerate, boilerplate) |
| 3 | Gemini 2.5 Flash | ★★★ | Cheap | Bulk doc generation, 200k+ file scans |
| 4 | GPT-5.3-Codex | ★★★ | Mid | Terminal-Bench 2.0: 77.3% — shell/CLI specialised tasks |

**Terminal tasks specifically:** Codex outperforms Claude on Terminal-Bench 2.0 (Codex 77.3% vs Claude 65.4%). For shell-centric tasks where reasoning isn't the bottleneck, a cheap Codex model often beats a premium Claude model on quality *and* costs less from the OpenAI quota.

**Multimodal / live web:** Only Gemini has native multimodal reasoning and built-in Google Search grounding. No Claude or Codex model matches this for screenshot analysis, diagram reasoning, or tasks requiring fresh web data.

### Cost/quality sweet spots

| Task shape | Best option | Why |
|------------|-------------|-----|
| Architecture / risky decisions | Claude Opus 4.7 (in-process) | Free on Max; best reasoning |
| Feature implementation, scoped fixes | Claude Sonnet 4.6 (in-process) | Free on Max; good balance |
| Mechanical edits, file shuffling | Claude Haiku 4.5 (in-process) | Free on Max; fast |
| Second-opinion code review | Codex GPT-5.5 at `medium` effort | Different model perspective; `gpt-5.4` is cheaper if quality is sufficient |
| Shell/terminal-centric task | Codex lightweight model (`spark` or equivalent) | Terminal-Bench advantage; cheap |
| 50k+ line repo summary | Gemini 2.5 Flash | $0.30/MTok in; 1M context; output is small → good cost shift |
| Large-context analysis with reasoning | Gemini 2.5 Pro (≤200K) | $1.25/MTok in; better than Flash for nuanced output |
| Screenshot / diagram / visual | Gemini 3.1 Pro Preview (or current flagship) | Only multimodal option |
| Live web data mid-analysis | Gemini flagship | Built-in Search grounding |

---

## 4. Maintenance

**Trigger to update this doc:** any of these events warrants a fresh fetch + doc update:
- A new Claude / GPT / Gemini model GAs or enters public preview
- An existing model is retired or repriced
- A CLI releases a new version that changes default models or flag names
- This doc's last-updated date exceeds **90 days** (flagged by `instruction-audit` drift scan)

**How to update — always fetch live, never use training memory:**

```
Fetch these pages and update the tables in §2:
  Anthropic:  https://www.anthropic.com/pricing
  OpenAI:     https://openai.com/api/pricing/
  Google:     https://ai.google.dev/pricing

Verify CLI flag→model mappings:
  codex --help          (or: codex exec --help)
  gemini --help

Check benchmark leaderboards for coding quality ordering (§3):
  SWE-bench:       https://www.swebench.com/
  Terminal-Bench:  search for "Terminal-Bench" on GitHub/arXiv

Then update:
  - The pricing tables in §2
  - The quality ordering table in §3 if rankings have materially shifted
  - The "Last updated" date at the top of this file
  - The §4 per-CLI model tables in global-skills/cross-agent-delegate/SKILL.md
    (model names, cost tiers, and the spark/flash alias footnotes)
```

**What NOT to do:** Do not update this file by asking an AI assistant to recall prices from training data. Model pricing changes frequently and training data is always stale. Fetch the pages above and use the live figures.

**Ownership:** maintained by the `instruction-audit` drift check (item 8) and the §11 maintenance checklist in `cross-agent-delegate`. Anyone running `/instruction-audit drift` is responsible for flagging an overdue update.
