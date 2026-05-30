---
description: Heavy product inception conversation across four lenses (business, user, product, technical). Composes with built-in /init for stack scaffolding, then conducts the deep intake. Produces STRATEGY.md, ARCHITECTURE.md, ROADMAP.md. Use --quick for a 6-question fast path producing only STRATEGY.md.
argument-hint: "[optional: brief project description] [--quick]"
role: orchestrator
---

# Init (product inception)

You are the **orchestrator**. The `intake-author` conducts the inception conversation; you compose with built-in `/init` and manage the flow.

This is the heaviest command in the workflow. A deep intake takes 30-60 minutes of real back-and-forth. That's appropriate — this is where the leverage is.

## Parse arguments first

Check `$ARGUMENTS` for `--quick`:

- **Default (deep mode)**: full four-lens conversation. Produces STRATEGY.md, ARCHITECTURE.md, ROADMAP.md.
- **`--quick` flag present**: 6-question fast path. Produces STRATEGY.md only. ARCHITECTURE.md and ROADMAP.md skipped — user runs `/sb-init` without `--quick` when ready.

If `--quick` is set, surface what's being skipped before starting: "Quick mode: this produces STRATEGY.md only. The technical architecture and roadmap conversations are skipped — re-run `/sb-init` without `--quick` when ready. Proceed?"

## Procedure

### Step 1: Pre-check

Check if `docs/STRATEGY.md` exists. If yes, this is a refinement — confirm with the user: overwrite, augment, or cancel?

### Step 2: Run built-in /init first

Tell the user: "I'll first run Claude Code's built-in `/init` to scan your codebase and scaffold CLAUDE.md. Then we'll do the inception conversation on top of what it generates."

Invoke built-in `/init`. It scans `package.json`, source layout, etc., and scaffolds CLAUDE.md with stack and conventions. Surface its output briefly.

If built-in `/init` is unavailable (older Claude Code) or fails, tell the user and continue — intake-author will handle stack questions itself.

### Step 3: Dispatch intake-author

Dispatch with mode (deep or quick) and any user-provided input (PDF path, project description, etc.):

```
Mode: <deep | quick>
Built-in /init has run: <yes | no>
User-provided input: <PDF path, description, or "none">

Conduct the intake per your agent specification. One question at a time.
```

The author conducts the conversation. **This is interactive — don't rush.** The user processes one question at a time better than a survey. If the user asks to skip a question, let them; write `<TODO>` and move on.

### Step 4: Surface intake outputs (draft)

When the author returns DONE, surface the file list as a **draft**:
- **Deep mode**: STRATEGY.md, ARCHITECTURE.md, ROADMAP.md
- **Quick mode**: STRATEGY.md only, plus a note that ARCHITECTURE.md and ROADMAP.md were skipped

Show any `<TODO>` markers so the user knows what's incomplete.

Tell the user: "Draft is ready. Two independent reviews next (different model than the intake author, for blind-spot diversity), then your approval."

### Step 5: Dispatch inception-auditor

Dispatch the `inception-auditor` agent. It runs on **sonnet** while intake-author runs on **opus** — the deliberate model diversity means the audit isn't laundered through the same blind spots that shaped the draft.

```
Mode: inception
Target docs: docs/STRATEGY.md, docs/ARCHITECTURE.md (if present), docs/ROADMAP.md (if present)
Research grounding (if present): docs/research/INDEX.md and each topic's INSIGHTS.md
```

Pass file paths only — no pre-digestion of what the docs say. Fresh eyes is the whole point.

When the auditor returns, SURFACE its verdict (`ship | revise | rewrite`) and the blocking + non-blocking findings.

- `ship` → proceed to Step 6.
- `revise` → present blocking findings to the user. Two options:
  - (a) Re-dispatch `intake-author` with the findings to address (another conversational pass on the specific gaps). Then re-audit.
  - (b) User accepts the gaps and overrides — log the override to `docs/decisions/<date>-inception-override.md` with reasoning. Proceed.
- `rewrite` → surface to user. Recommend: stop `/sb-init`, optionally run `/sb-research` on the under-grounded domains, then re-run `/sb-init` from scratch. Don't continue patching a fundamentally weak intake.

### Step 6: Dispatch devil-advocate (strategy target)

Dispatch `devil-advocate` with `Target: strategy`. Also runs on **sonnet** for the same model-diversity reason. It argues against the *bet itself* — not execution.

```
Target: strategy
Read: docs/STRATEGY.md, docs/ARCHITECTURE.md (if present), docs/ROADMAP.md (if present), docs/research/*/INSIGHTS.md
```

When devil-advocate returns, SURFACE the case-against. The verdict (`defer | proceed-with-caveats | ship`) is advisory — the user decides.

This stage is **not** skipped in `--quick` mode. Even with only STRATEGY.md, the bet itself is worth one independent argument-against.

### Step 7: Final approval

Present the consolidated picture:

```
Documents:           <list>
Inception audit:     <verdict, key findings>
Devil's advocate:    <verdict, top 1-2 concerns>
```

Ask the user: **accept, revise, or stop?**
- **accept** → suggest next step:
  - After deep mode: `/sb-spec <run-id>` (default — autonomous loop), or `/sb-plan <feature>` (manual lane on a single feature from ROADMAP.md Horizon 1).
  - After quick mode: run `/sb-init` again without `--quick` when ready, OR proceed to `/sb-plan` with a clear feature in mind.
- **revise** → loop back to Step 5 (re-dispatch intake-author with concerns to address, re-audit, re-advocate).
- **stop** → leave docs in place. The user resumes by re-running `/sb-init` later.

Do not auto-trigger `/sb-spec` or `/sb-plan`.

## Rules

- You don't conduct the intake. The author does. Don't shortcut it by asking questions yourself.
- Don't auto-trigger `/sb-spec` or `/sb-plan`. Let the user read the generated docs and audit findings first.
- If the user resists the deep intake entirely ("just let me code"), offer `--quick` as the formal lightweight path rather than informally skipping questions in deep mode.
- **Never skip the inception audit or strategy devil-advocate.** Even in `--quick` mode they run. The whole point of model-diversity audit is that the author can't catch what the author missed; skipping it because "the conversation went well" reintroduces the exact failure mode the audit exists to prevent.
- **The audit runs on a different model than the author by design.** If the model spec on `inception-auditor.md` or `devil-advocate.md` somehow matches the author's model, surface the misconfiguration to the user — don't silently proceed.

## Handling escalations from the author

If the author returns `STATUS: ESCALATE`, surface the packet exactly:
- Lead with QUESTION
- Show OPTIONS and RECOMMENDATION
- Wait for the user's answer
- Log to `docs/decisions/<date>-<slug>.md`
- Resume the author with the user's decision

Do not invent answers. Do not summarize options into something prettier — the user is the team lead and needs the raw packet.
