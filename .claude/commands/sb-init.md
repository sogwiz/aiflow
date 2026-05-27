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

### Step 4: Surface outputs

When the author returns DONE, surface the file list:
- **Deep mode**: STRATEGY.md, ARCHITECTURE.md, ROADMAP.md
- **Quick mode**: STRATEGY.md only, plus a note that ARCHITECTURE.md and ROADMAP.md were skipped

Show any `<TODO>` markers so the user knows what's incomplete.

### Step 5: Suggest next step

- After deep mode: suggest `/sb-plan <first feature from ROADMAP.md Horizon 1>`
- After quick mode: suggest running `/sb-init` again without `--quick` when ready, OR proceeding to `/sb-plan` with a clear feature in mind

## Rules

- You don't conduct the intake. The author does. Don't shortcut it by asking questions yourself.
- Don't auto-trigger `/sb-plan`. Let the user read the generated docs first.
- If the user resists the deep intake entirely ("just let me code"), offer `--quick` as the formal lightweight path rather than informally skipping questions in deep mode.

## Handling escalations from the author

If the author returns `STATUS: ESCALATE`, surface the packet exactly:
- Lead with QUESTION
- Show OPTIONS and RECOMMENDATION
- Wait for the user's answer
- Log to `docs/decisions/<date>-<slug>.md`
- Resume the author with the user's decision

Do not invent answers. Do not summarize options into something prettier — the user is the team lead and needs the raw packet.
