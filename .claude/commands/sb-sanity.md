---
description: Drift check — fresh-eyes audit of alignment across STRATEGY, ARCHITECTURE, ROADMAP, plans, code, and CLAUDE.md. Use after context compaction, after long sessions, or when picking up a project after a gap. Can optionally produce docs/HANDOFF.md as orientation snapshot.
argument-hint: "[--with-handoff]"
role: orchestrator
---

# Sanity (drift check)

You are the **orchestrator**. You don't perform the check — the whole point is independent verification by a fresh agent. Dispatch the `auditor` in drift mode with completely fresh context.

## Critical: no pre-digestion

When dispatching the auditor, hand it **file paths only** and the task. Do not paste your interpretation of what the project is or should be. If you summarize the project for it, you transfer your blind spots and the check becomes worthless. The whole value is that it reads cold.

## Parse arguments

Check `$ARGUMENTS` for `--with-handoff`:

- **Default**: drift check only. Produces audit findings.
- **`--with-handoff`**: also produces `docs/HANDOFF.md` orientation snapshot — useful when you're picking up a project after a gap and want both the alignment check and the orientation doc in one go.

## Procedure

1. **Locate the docs:**
   - `docs/STRATEGY.md` (must exist)
   - `docs/ARCHITECTURE.md` (must exist if running deep mode; absent in `--quick`-initialized projects)
   - `docs/ROADMAP.md` (similar)
   - `docs/plans/` (read 3-5 most recent)
   - `docs/decisions/` (read 10-20 most recent)
   - `CLAUDE.md` (must exist)
   - Code at project root

   If `STRATEGY.md` or `CLAUDE.md` is missing, tell the user: "Sanity check needs STRATEGY.md and CLAUDE.md to anchor the audit. Run `/sb-init` first."

   If ARCHITECTURE.md or ROADMAP.md is missing (quick-init project), tell the user: "This project was initialized with `/sb-init --quick`, so ARCHITECTURE.md and ROADMAP.md don't exist. Drift check will run against the artifacts that do exist, but checks A and B (strategy-to-architecture, architecture-to-roadmap) will be skipped. Run `/sb-init` without `--quick` to enable full drift checking."

2. **Dispatch the auditor in drift mode.** Use this exact dispatch text — no project summary, no commentary:

   ```
   You are doing a Mode 3 (drift) audit. Read the project with completely fresh eyes —
   do not assume anything from what I tell you here.

   Paths:
   - Strategy: docs/STRATEGY.md
   - Architecture: docs/ARCHITECTURE.md (if exists)
   - Roadmap: docs/ROADMAP.md (if exists)
   - Recent plans: docs/plans/ (5 most recent)
   - Recent decisions: docs/decisions/ (20 most recent)
   - Conventions: CLAUDE.md
   - Code: project root

   Produce a drift report per Mode 3 of your agent specification.

   produce_handoff: <true | false>
   ```

3. **Read the auditor's report.** Surface findings to the user grouped by the five alignment checks (STRATEGY→ARCHITECTURE, ARCHITECTURE→ROADMAP, ROADMAP→Plans, Plans→Code, Code↔CLAUDE.md). Do not silently fix things.

4. **On SIGNIFICANT DRIFT, escalate.**
   - Surface findings and the top-3 fix list
   - Ask the user: which way should truth flow? Update docs to match reality, change code to match docs, or revise strategy/roadmap because reality has moved on?
   - Log the resolution to `docs/decisions/<date>-drift-resolution.md`
   - Do not auto-apply

5. **On MINOR DRIFT, surface but don't escalate.** List discrepancies; offer to fix any the user wants addressed.

6. **On ON TRACK, say so and stop.** A clean report is the right answer most of the time.

7. **If `--with-handoff` was set**, the auditor also wrote `docs/HANDOFF.md`. Surface its location and a brief note: "Orientation snapshot also written to docs/HANDOFF.md — useful starting point for anyone picking this up."

## When to run this

- After context compaction in the main session (you doubt your own context)
- After a long session where you've made many decisions
- Before starting major new work (verify the foundation)
- When the main agent contradicts itself or seems to have forgotten earlier decisions
- Picking up a project after a gap — use `--with-handoff` to get both the alignment check and orientation snapshot

## Why this replaces both /sb-handoff and /sb-resume from earlier versions

Earlier versions had three separate commands: drift check (sanity), alignment test (handoff), orientation snapshot (resume). In practice they all share the same core: a fresh agent reading the project cold and producing findings. The differences were in *what they surfaced* and *whether they wrote a snapshot doc*.

This command does both:
- The drift audit covers what `/sb-handoff`'s alignment test used to check (does the project's trajectory make sense given the docs).
- The `--with-handoff` flag produces what `/sb-resume` used to produce (orientation snapshot).

One command instead of three. Same coverage. Easier to remember.

## Rules

- **Never paste your own summary of the project into the auditor's dispatch.** Non-negotiable. The test is worthless if the auditor inherits your context.
- **Don't auto-fix anything.** Surface findings; let the user direct remediation.
- **Trust the auditor's verdict even when it surprises you.** Surprise is the point — if the auditor says drift and you "know" there isn't any, that's exactly the case where the auditor is more right than you.
