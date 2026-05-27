---
description: Code review with four sequential perspectives (correctness, edge cases, security, simplicity) in one agent. Lighter than four parallel reviewers — but with explicit frame-switching to keep findings sharp.
argument-hint: "[optional: git ref range; defaults to recent changes]"
role: orchestrator
---

# Review

You are the **orchestrator**. Dispatch the `reviewer` agent.

Diff target: **$ARGUMENTS** (default: unstaged + staged changes, fall back to `HEAD~1..HEAD`)

## Procedure

1. Confirm the diff target. Print which files are in scope so the user can correct if wrong.
2. Dispatch `reviewer`. Pass:
   - The diff range
   - Pointer to the source plan in `docs/plans/`
   - "Run all four passes (correctness, edge cases, security, simplicity). Switch frame explicitly between passes."
3. The reviewer writes consolidated findings to `docs/reviews/<date>-<slug>-review.md`.
4. Surface to the user. Offer to apply fixes for blocking findings (confirm before editing).
5. After fixes, optionally re-dispatch for the affected passes to confirm closure.

## On escalation

If the reviewer escalates (e.g., "implementation is correct but plan missed X"), surface the packet. The user decides whether to proceed with the review or pause for plan revision.

## Why one reviewer agent, not four

The earlier four-parallel-reviewer pattern is genuinely good but heavier. This template trades some independence for simplicity:
- One agent, four passes, sequential
- Explicit frame-switching between passes (the agent is instructed to mentally close one frame before opening the next)
- Coverage stays good for typical PRs; for large/risky changes, consider running the heavier four-parallel pattern manually (you can copy/adapt from the v3 compound-agents template)

## Rules

- The user can ask for a partial review ("just security on these auth changes") — pass that to the reviewer as a single-pass instruction.
- After review:
  - **ship** → suggest committing
  - **ship with fixes** → offer to apply, then re-review affected passes
  - **do not ship** → surface findings; user decides whether to revise plan or revise code
