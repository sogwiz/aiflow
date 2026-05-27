---
description: Post-ship evaluation against named metrics. Default window is 30 days (the one that catches "we built the wrong thing"). Use --window 7d for a quick shipping-quality check.
argument-hint: "<feature slug> [--window 7d|30d]"
role: orchestrator
---

# Eval

Structured conversation. No agents dispatched — eval is best done with full context and user honesty, not isolated agent work.

Feature: **$ARGUMENTS**

## Procedure

1. **Find the feature.** Match the slug against `docs/plans/`. Read the plan, source requirements, review, STRATEGY.md.

2. **Determine window.** Default is 30 days (the premise check — did we build the right thing). If `--window 7d` is specified, do the shipping-quality check instead.

3. **Pull the metrics.** Read `STRATEGY.md#metrics` (project) and the plan's `#metrics` section (feature). List them with targets.

4. **Ask the user, metric by metric.**

   For each metric:
   - What's the actual value? (point to where it should be instrumented per the plan)
   - How does it compare to target? Hit / partial / missed / no-data.
   - If "no-data": instrumentation promised in the plan isn't producing measurable output. That's its own finding regardless of feature outcome.

5. **For 30-day window**, additionally ask:
   - Did this trigger any red lines?
   - Did the devil's advocate's case turn out to have merit?
   - What's the maintenance load?
   - Are there learnings to encode into CLAUDE.md or agent prompts?

   **For 7-day window**, additionally ask:
   - Did it ship as planned, or were there deviations?
   - What broke? Bugs, edge cases, missed cases?
   - Did review catch things that should have been caught at planning?

6. **Write the eval** to `docs/evals/<feature-slug>-<window>.md`:

```markdown
# Eval: <feature> — <window>

_Shipped: <date>_  _Evaluated: <date>_

## Metrics

| Metric | Target | Actual | Status | Source |
|---|---|---|---|---|
| <name> | <target> | <value> | hit / partial / miss / no-data | STRATEGY#metrics or plan#metrics |

## Outcome
<Hit / partial / missed / too early to tell — with evidence.>

## What worked / what broke
<honest list>

## Gaps the system missed
<things requirements-audit, plan-audit, concerns-audit, or devil's-advocate should have caught>

## Learnings for next cycle
<specific, encodable>

## Verdict
<Keep | iterate | sunset>

## Doc freshness check
<see below>
```

7. **Run the freshness pass.** Load the `doc-freshness` skill. The skill checks:
   - Does STRATEGY.md#metrics still name the right metrics?
   - Does STRATEGY.md#user still match who's actually using this?
   - Does CLAUDE.md need updating based on this session's work?
   - Are there patterns or contradictions in `docs/decisions/`?

   The skill returns "no updates needed" or specific proposed diffs. Surface to user; they approve or skip each.

8. **Surface patterns across evals.** After 3+ evals, look for recurring gaps. Those are the highest-signal inputs for tuning agents or promoting conventions to CLAUDE.md.

## Why no agent for evals

Evals are subjective — they depend on user judgment about success. An adversarial auditor on an eval would just nitpick interpretation. The honest version is the user being structured-honest with themselves, with this command providing the structure.

## Rules

- **Be honest about misses.** An eval that says "everything worked" is suspect; either the feature was trivial or you're not looking hard.
- **Specific evidence beats vibes.** "Users seem happy" is not data; "5 of 7 pilot users used it twice in week 1" is.
- **The 30-day eval is where premise problems surface.** If you only ever run 7-day, you'll catch bugs but miss whether you built the right thing.
