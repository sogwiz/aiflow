---
description: Execute the most recent approved plan. Single agent, full context. Stops if reality diverges from the plan.
argument-hint: "[optional: path to plan; defaults to most recent in docs/plans/]"
role: orchestrator
---

# Work

This is a thin orchestrator — execution is best done in one context with full visibility. There's no dedicated implementor agent for code; you (the main session) do the work directly, but under strict rules.

Plan: **$ARGUMENTS** (default: most recently modified `docs/plans/*.md`)

## Procedure

1. Confirm the plan has been through the full `/sb-plan` pipeline. Check that audit files exist (`docs/audits/*-<slug>-*`). If not, warn: "You're executing a plan that wasn't audited. Proceed?"
2. Read the plan in full. Read `CLAUDE.md` for conventions. Read `docs/decisions/` for any precedent that affects this work.
3. For each step in the plan's `## Steps` section:
   - State which step you're on
   - Make the change
   - Run the relevant test or verification
   - If you hit a decision, route it:
     - Tactical → decide silently
     - Technical → decide, then log it under the plan's `## Decision log` or `## Deviations` with rationale
     - Architectural / strategic / high-risk → stop and ask the user; log the resolved decision to `docs/decisions/`
   - **If reality diverges from the plan** (file shape different than expected, dependency missing, function doesn't exist), **stop**. Do not improvise. Pause and ask:
     - Revise the plan to match reality?
     - Proceed with a documented deviation?
     - Abort and re-plan?
4. Honor the `## Tests` section. Run them. Do not declare done until passing.
5. When all steps complete, print a **mandatory divergence summary**. This is required even if there were no divergences — explicitly say so. The summary becomes a forcing function for honest reporting that `/sb-review` will check.

   **Format:**
   ```
   ## Divergence summary

   **Divergences encountered:** <number>

   <For each divergence:>
   - **Step #<n> / file `<path>`:** Plan said <X>, reality was <Y>. Resolution: <what I did and why>. Logged to plan §Deviations: <yes / no — and if no, why not>.

   <If no divergences:>
   - None. All steps matched the plan exactly. Code in `<files>` matches what the plan specified.
   ```

   Also produce a final summary:
   - Files changed
   - Technical decisions logged
   - Tests run + results
   - Followups noticed but not in scope (logged under `## Followups`)

## Rules

- No scope creep. If you notice unrelated issues, add them to the plan's `## Followups`. Do not fix them in this session.
- No skipping tests because "obviously correct."
- Technical decisions can be made without pausing, but they must be logged. Architectural, strategic, and high-risk decisions require user approval.
- **The divergence summary is mandatory.** If you find yourself wanting to omit it because "nothing meaningful diverged," produce it anyway saying so explicitly. Silent matches and silent improvisations look identical from outside — the summary is what distinguishes them.
- If you have to deviate from a step, record it in the plan file under `## Deviations` with rationale. The summary then references that section.
- After completion, suggest `/sb-review` as the next step. The reviewer agent specifically checks for undocumented divergences (code that does things the plan didn't list, without a `## Deviations` entry explaining why).

## Why no dedicated implementor agent

Earlier versions of this template had a separate implementor subagent. In practice it added overhead without benefit — execution benefits from full plan context, decisions log context, and conversation history. The main session does the work, but under the discipline of an approved plan. The audit work already happened in `/sb-plan`; the review work happens in `/sb-review`. `/sb-work` is the disciplined middle.

## Escalation

The main session is *you* the LLM, talking to the user. There's no separate agent to escalate from. Instead, when in doubt:
- Stop
- Surface the question clearly
- Get explicit user input before proceeding
- Log non-trivial decisions to `docs/decisions/`
