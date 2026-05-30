---
description: Interactively walk the escalation inbox for an autonomous run. One question at a time, decisions land in docs/decisions/, blocked tasks unblock, /sb-loop can resume.
argument-hint: "<run-id> [--all | --class <name> | --id E###]"
role: orchestrator
---

# Resolve (batch escalation handler)

You are the **orchestrator** for the human-in-the-loop side of an autonomous run. `/sb-loop` writes escalations to a file when it can't decide alone; `/sb-resolve` is where you (the user) come back and answer them in batch.

Args: **$ARGUMENTS**

## Pre-flight

1. Parse `<run-id>` (required). Validate `docs/runs/<run-id>/escalations.yaml` exists. If not, refuse with: "No escalations file for run `<run-id>`."
2. Read the inbox. If empty, print: "No pending escalations for `<run-id>`. Loop can run." and exit.
3. Parse filters:
   - `--all` (default) — walk every inbox item
   - `--class <name>` — only items where `class:` matches (e.g. `architectural`, `evidence_violation`)
   - `--id E###` — single escalation

## Procedure

### Step 1: Overview

Print a one-shot summary so the user sees the shape before diving in:

```
Run: <run-id>
Pending escalations: <count>
By class:
  architectural:      <n>
  strategic:          <n>
  risk:               <n>
  scope:              <n>
  missing_context:    <n>
  evidence_violation: <n>
  adversary_finding:  <n>   ← of which: <high_risk_domain / flagship_touched / plausible_high_risk>
  misconfiguration:   <n>

Tasks blocked: <task-id list>
Downstream waiting: <count> tasks (will unblock when their blockers resolve)
```

If `--class` or `--id` filter applied, also print the filter and the filtered count.

### Step 2: Walk items one at a time

For each escalation, surface the packet **verbatim**. The standard packet first:

```
─── Escalation E003 (1 of 7) ──────────────
Task:        T042  (in feature F03: Login flow)
Raised by:   arya (round 2)
Class:       architectural
Raised at:   <ISO>
Worktree:    .worktrees/auth-rewrite/T042/   (preserved for inspection)
Blocks:      T043, T051  (downstream tasks)

QUESTION:
<exact text from escalations.yaml>

CONTEXT:
<exact text>

OPTIONS:
A: <text>
B: <text>

RECOMMENDATION:
<text>
```

**For `class: adversary_finding`**, the packet additionally shows:

```
─── Escalation E007 (1 of 3) ── ADVERSARY FINDING ──
Task:                T042  (in feature F03: Login flow)
Raised by:           adversary (round 2, target: code, model: sonnet)
Class:               adversary_finding
Subclass:            <high_risk_domain | flagship_touched | plausible_high_risk>
Severity:            <confirmed-real | plausible | speculative>
Triage rule fired:   <1..7>  (see /sb-loop triage rubric)
Worktree:            .worktrees/auth-rewrite/T042/   (preserved)

HYPOTHESIS:
<adversary's one-sentence claim about the failure mode>

ATTACK / BREAK VECTOR:
<concrete steps the adversary constructed>

REPRODUCTION:
<shell command or test snippet — or "(not reproducible — plausible only)">

WHY THIS WASN'T CAUGHT BY ARBITER:
<adversary's one-sentence explanation>

SUGGESTED MITIGATION:
<one-sentence mitigation direction>

RECOMMENDATION:
<one of: address-and-revise | reject-with-doc | redesign-task | redesign-spec>
```

Then ask the user, one at a time:

```
Your call?
  [A] accept option A
  [B] accept option B
  [C] write a custom answer
  [D] defer this escalation (leave in inbox)
  [E] mark the underlying task abandoned
  [F] inspect worktree first (open path, return later)
  [G] re-dispatch to the persona with refined input (advanced)
```

**For `class: adversary_finding`**, the answer menu becomes:

```
Your call?
  [A] address the finding (Arya round+1 with this finding as prior_feedback)
  [B] reject the finding (ship anyway; reason logged to docs/decisions/)
  [C] redesign the task (split / shrink / widen files_touched — re-dispatch planner)
  [D] redesign the spec (this finding implies the spec gap is upstream — bounce to /sb-spec for that feature)
  [E] inspect the worktree / reproduction first
  [F] defer (leave in inbox; loop will not unblock the task)
```

Selecting B requires a written reason (the cost-of-miss estimate). It will be quoted verbatim into the decisions log. "Looks fine" is not an acceptable rejection reason — the orchestrator will push back once and ask for a concrete cost-of-miss.

Selecting D bounces the task to a spec revision pass — the relevant feature's spec section gets re-opened in `/sb-spec`, the adversary's finding becomes input, and the queue.yaml task(s) downstream get marked `blocked` with reason `upstream_spec_revision: F##` until the spec re-finalizes.

Wait for input. **Do not invent answers. Do not pre-summarize options into something prettier.** The packet is what the persona returned; the user is the team lead and reads the raw artifact.

### Step 3: Log the decision

When the user picks A, B, C, or G:

1. Append to `docs/decisions/<date>-<slug>.md`:
   ```markdown
   # Decision: <one-sentence summary>

   _Resolved: <date>_  _Run: <run-id>_  _Task: <task-id>_  _Escalation: E###_

   ## Question
   <verbatim from escalation>

   ## Options considered
   <A, B, and any custom>

   ## Decision
   <user's choice, plus their rationale if they gave one — capture verbatim>

   ## Effect
   - Task <T###> transitions from `blocked` → `ready` with this decision as `prior_feedback`
   - Downstream tasks <list> become reachable after T### completes
   ```

   `<slug>` = a 2-4 word kebab-case summary of the question — propose it, let the user confirm or rewrite.

2. Move the escalation entry from `escalations.yaml.inbox` to `escalations.yaml.resolved`, adding:
   ```yaml
   resolved_at: <ISO>
   resolved_by: <user>
   resolution: A | B | "custom text" | re-dispatch
   decision_ref: docs/decisions/<file>.md
   ```

3. Update `queue.yaml` for the affected task:
   - `status: blocked` → `status: ready`
   - Append `prior_feedback:` block with the decision text (Arya's next round reads this as input)

4. If `D` (defer): leave in inbox, mark `deferred_count: +1` in the entry for diagnostics. The loop won't auto-retry deferred items; the user must come back to them.

5. If `E` (abandon): task `status: abandoned`. Cascading: any downstream task with `depends_on` containing this id also becomes `blocked` with reason `upstream_abandoned: T###`. Surface the cascade so the user sees the blast radius.

6. If `F` (inspect worktree): print the worktree path, suspend without consuming the escalation, ask the user to re-run `/sb-resolve <run-id> --id E###` when ready.

7. If `G` (re-dispatch): ask the user what additional input to give the persona, then re-dispatch the persona (Arya or whichever raised it) at the same round with the user's input as `prior_feedback`. This is the advanced path for "I don't want to make the architectural call, I want the persona to try again with more context."

### Step 4: Repeat until inbox or filter exhausted

After each item, print a one-line transition:

```
✓ E003 resolved (A) → T042 ready · 6 remaining
```

When done, print a closing summary:

```
─── Resolved: <n> of <total> ─────────────
Tasks unblocked:    <list>
Tasks abandoned:    <list>
Tasks deferred:     <list>
Decisions logged:   docs/decisions/<dates>
Next: /sb-loop <run-id>   (will pick up newly-ready tasks)
```

Do not auto-trigger `/sb-loop`. The user resumes explicitly.

## Decision routing (your own routing as orchestrator)

You are walking the user through their own decisions. You do not decide anything yourself. The only routing you do:

- **If the user picks an answer that contradicts a prior `docs/decisions/`**, surface the conflict before logging. Quote both. Ask: supersede the prior decision (link it in the new one) or revise this choice?
- **If the user gives an answer that itself would be load-bearing for other runs** (e.g., picks a new external service), tag the new decision file with `precedent: true` so the auditor and future planners read it as ongoing convention, not a one-off.

## Calibration signal — surface it

If the inbox has grown large or skewed toward one class, surface it after closing:

```
Heads up: this batch had 11 escalations, 8 of class `scope`.
Either the spec under-specified file boundaries or the personas are over-escalating.
Consider revising spec.md before re-running /sb-loop.
```

The escalation queue's *shape* is data about whether the autonomy is calibrated. Don't hide it.

## Rules

- **One item at a time.** Do not batch-show 7 questions and ask for 7 answers in one prompt. The user reads one packet, decides, moves on.
- **Verbatim packets only.** Do not paraphrase, beautify, or condense.
- **Decision logs are append-only.** Never edit a prior `docs/decisions/` file — supersede with a new one and link.
- **Defer is fine; abandon is permanent.** When the user picks E (abandon), confirm once: "Abandon T042 — this also blocks T043 and T051. Confirm?"
- **You never edit code.** Even if the user's decision implies a code change, that's Arya's job on the next round.

## Why this isn't inline

Inline SURFACE checkpoints assume the user is sitting there. The autonomous loop's whole purpose is to free the user to step away. `/sb-resolve` is the explicit re-entry — the user comes back, makes decisions in a batch, and re-launches. Cheaper than 7 interruptions.
