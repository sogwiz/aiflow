---
name: doc-freshness
description: Use when a session may have changed something that documents in CLAUDE.md, STRATEGY.md, or the decisions log should reflect. Triggers include the stop-hook at session end, /sb-eval's freshness pass, or any user phrase like "did anything change worth documenting", "update the docs", "is anything stale", or "should we update CLAUDE.md". Loads only when triggered — does not consume context otherwise.
---

# Doc freshness

You're checking whether docs in the repo reflect what just happened. The goal is to surface *specific* proposed edits, not to do an exhaustive audit. If nothing changed, return early — that's the right answer most of the time.

## What to check, in order of value

### 1. CLAUDE.md conventions vs. session activity
The highest-value check. If this session:
- Introduced a new test pattern that should be the project default → propose CLAUDE.md edit
- Settled on a naming convention by repetition (e.g., five new files all use `kebab-case-handlers.ts`) → propose CLAUDE.md edit
- Added a new tool, command, or library that becomes a project-wide expectation → propose CLAUDE.md edit
- Documented a "we always do X" decision verbally that has no written home → propose CLAUDE.md edit

Don't propose updates for one-off choices — only for things that should apply to future work.

### 2. Plan deviations that suggest plan-template drift
If a plan in `docs/plans/` accumulated a `## Deviations` section during execution, ask: was the deviation because of (a) the plan being wrong for this case, or (b) the plan *template* missing something every plan needs? Only (b) is a template fix.

Example: deviation says "step 5 didn't account for the existing rate limiter." That's case (a) — one plan was incomplete. No template fix.

Example: deviations across three different plans all say "had to add Sentry instrumentation that wasn't in the plan." That's case (b) — the plan template should require an instrumentation step. Propose updating `planner.md`.

### 3. STRATEGY.md vs. accumulated reality
If `/sb-eval` triggered this skill, check:
- Are metrics in `STRATEGY.md#metrics` still the right ones? Or have evals consistently shown we should be tracking something different?
- Does the primary user in `STRATEGY.md#user` still match who's actually using the product?
- Have any red lines been violated quietly without revising the strategy?

Don't propose strategy edits from a single eval — wait for a pattern across at least two evals before suggesting a strategic shift.

### 4. Decisions log: contradictions and patterns
Scan `docs/decisions/` for:
- **Patterns**: 3+ decisions resolving the same kind of question the same way → candidate for CLAUDE.md promotion (so the question stops being asked)
- **Contradictions**: a recent decision contradicts an older one → surface the inconsistency to the user; they need to pick which holds

This is the same check `/sb-eval` does in its freshness pass. If both are running, don't duplicate; the eval already covers it.

## Output format

If nothing needs updating, return exactly:

```
No doc updates proposed. Checked: CLAUDE.md, recent plan deviations, STRATEGY.md, decisions log.
```

If updates are proposed, return:

```markdown
## Proposed doc updates

### CLAUDE.md
**Section:** <which one>
**Reason:** <what happened in this session that motivates this>
**Diff:**
- <old text or "(new section)">
+ <new text>

### STRATEGY.md
<same shape, if applicable>

### Decisions log
<contradictions or patterns to surface, if applicable>
```

## Rules

- **Be specific.** "CLAUDE.md should be updated" is not actionable; "Add to CLAUDE.md#conventions: 'New handler files use kebab-case (established this session by files X, Y, Z)'" is.
- **Propose, don't apply.** Always surface the diff to the user for approval. The system never silently rewrites docs.
- **Resist over-proposing.** If you find yourself proposing more than 3 updates in one pass, you're inventing work. Pick the most important and skip the rest.
- **Single occurrence is anecdote, two is pattern.** Don't suggest convention changes from one data point.
- **Default to "no updates."** If you can't make a specific, concrete proposal, return the "nothing to update" response. False positives train users to ignore the skill.

## Why this skill exists rather than baking the logic into commands

The check matters most at session end, when you have full context on what just changed. Putting it in a skill (loaded only when triggered) keeps it out of context for sessions that don't need it. Putting it behind a stop-hook makes it deterministic — no remembering to run it — without forcing it on users who turn the hook off.
