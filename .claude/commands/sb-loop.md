---
description: Default execution mode. Autonomous scheduler over a named run — Arya → Crucible → Arbiter per task, parallel where dependencies allow, async escalations, resumable. No SURFACE checkpoints in the inner loop.
argument-hint: "<run-id> [--max-concurrent N] [--max-rounds N] [--task T###] [--resume]"
role: orchestrator
---

# Loop (autonomous execution)

You are the **orchestrator** for an autonomous run. You schedule tasks off `docs/runs/<run-id>/queue.yaml`, dispatch the persona graph per task, update state on every transition, and surface progress + escalations to the user.

This is the **default execution mode**. The per-feature `/sb-plan` → `/sb-work` flow is the manual lane for one-off changes that don't justify a full run.

Args: **$ARGUMENTS**

## Pre-flight

1. Parse `<run-id>` (required) and flags:
   - `--max-concurrent N` (default 4) — concurrency cap across the whole run
   - `--max-rounds N` (default 3) — dev↔tester loop cap per task
   - `--task T###` — execute only this one task (debug mode)
   - `--resume` — explicit resume after a crash; default behavior already resumes, but this flag makes intent explicit and prints a resume diff
2. Validate flags:
   - `max_rounds = 3` when `--max-rounds` is absent
   - If present, `--max-rounds` must be an integer >= 1. If invalid, stop and ask for a valid value.
   - `max_rounds` counts Arya implementation attempts for a task. Round 1 is the first implementation; each Crucible FAIL or blocking Arbiter finding consumes the next Arya round.
3. Read `docs/runs/<run-id>/manifest.yaml`. If `status: pending_approval`, refuse and direct user to re-run `/sb-spec`. If `status: drained` or `completed`, ask whether to re-open.
4. Read `docs/runs/<run-id>/queue.yaml`. Validate schema (`schema_version`, required fields per task). If invalid, escalate to user — do not "auto-repair."
5. Read `docs/runs/<run-id>/escalations.yaml`. If `inbox` is non-empty, surface: "There are <N> unresolved escalations. Run `/sb-resolve <run-id>` first, or pass `--ignore-escalations` to proceed and queue more." Do not silently proceed.
6. Confirm `git status` is clean on the merge-base branch named in the manifest. If not, refuse — the loop assumes a clean base for worktrees.
7. Set manifest `status: running`.

## The persona graph (per task)

```
                           ┌──────────────────────────────┐
                           │  Orchestrator picks T###     │
                           │  Creates worktree (if cfg)   │
                           │  status: in_progress         │
                           └──────────────┬───────────────┘
                                          ↓
                              ┌───── Arya (round N) ─────┐
                              │   implements task         │
                              └────────────┬──────────────┘
                                ESCALATE   │  DONE
                                  │        ↓
                                  │   ┌── Crucible (round N) ──┐
                                  │   │  evidence-first then   │
                                  │   │  correctness tests     │
                                  │   └─┬────┬────────┬────────┘
                                  │     │    │ FAIL   │ BLOCKED
                                  │     │    │  → Arya round N+1
                                  │     │    │     (max_rounds cap)
                                  │     │    └──→ escalation
                                  │     ↓ PASS
                                  │   ┌── Arbiter (built-in reviewer) ──┐
                                  │   │ correctness / edge / security / │
                                  │   │ simplicity / evidence survival  │
                                  │   └─┬─────────────────┬─────────────┘
                                  │     │ ship | ship-w/f │ do-not-ship
                                  │     ↓                 ↓
                                  │   Adversary triggers? (diff≥50 / risk path /
                                  │   flagship_scenario / high-risk lane)
                                  │     │
                                  │     ├─ no  → merge, status: done
                                  │     │
                                  │     └─ yes →  ┌── Adversary (sonnet, code) ──┐
                                  │               │ generative failure scenarios │
                                  │               │ severities: confirmed-real / │
                                  │               │ plausible / speculative      │
                                  │               └─┬────────────────────────────┘
                                  │                 ↓
                                  │           Orchestrator triage (deterministic rubric)
                                  │                 │
                                  │                 ├─ ACCEPT          → Arya round N+1
                                  │                 ├─ REJECT-WITH-DOC → ship + log
                                  │                 └─ ESCALATE        → escalations.yaml
                                  │
                                  ↓
                              escalations.yaml.inbox << packet
                              status: blocked
```

## Scheduling loop

Run this loop until quiescence (queue drained OR remaining tasks all blocked/escalated):

```
while True:
    ready = tasks where status == 'ready' or status == 'todo'
        and all task.depends_on are status == 'done'
        and files_touched is disjoint with currently in_progress tasks
    if not ready and no tasks in flight:
        break   # quiescent

    take next min(max_concurrent - in_flight, len(ready)) tasks
    dispatch persona graph for each (parallel)
    on each completion, write status to queue.yaml, append to logs/, recompute ready

surface_summary()
set manifest.status = 'drained' if anything blocked, else 'completed'
```

**Parallelism rules:**

- Two tasks may run concurrently iff their `files_touched` sets are disjoint **and** their components are independent per `ARCHITECTURE.md`.
- Each in-flight task gets its own git worktree (`worktree: true` in defaults) under `.worktrees/<run-id>/<task-id>/`. Worktree is created at dispatch time, removed at task `done` (after merge) or `blocked` (preserved for inspection).
- Dispatch parallel agents in a single message — multiple `Agent` tool calls in one assistant turn. Await all returns before the next scheduling round.

**Single-task mode** (`--task T###`): skip the scheduler, run the persona graph for the one task, exit.

## Per-task persona graph (the inner loop)

For each picked task `T###`:

### 1. Setup

- Read the task from `queue.yaml`, the spec section from `spec.md#T###`.
- Determine if the task touches a **flagship scenario** (spec.md lists `flagship_scenarios` per feature; tasks reference scenario IDs).
- Create a worktree at `.worktrees/<run-id>/<task-id>/` branched from the manifest's `merge_base`.
- Set `queue.yaml` task status: `in_progress`, append start timestamp to log.

### 2. Arya round 1

Dispatch the `arya` agent with the inputs its spec requires (`run_id`, `task_id`, `queue_path`, `spec_path`, `log_path`, `worktree`, `round: 1`, `prior_feedback: ""`).

On return:
- `STATUS: DONE` → proceed to Crucible round 1
- `STATUS: ESCALATE` → write escalation packet to `escalations.yaml.inbox`, set task `status: blocked`, append blocker to log, **continue scheduling other tasks** (do not halt the loop)

### 3. Crucible round 1

Dispatch the `crucible` agent. Crucible enforces the **Evidence Contract gate** for flagship scenarios: visibility tests run **before** correctness tests. If the task touches a flagship scenario and the evidence contract is violated, Crucible returns `VERDICT: FAIL` with `failure_class: evidence_violation` — Arya's next round gets that block as `prior_feedback`.

On return:
- `VERDICT: PASS` → proceed to Arbiter
- `VERDICT: FAIL` → if `round < max_rounds`, dispatch Arya again at `round+1` with Crucible's FAIL block as `prior_feedback`. Loop back to step 3.
- `VERDICT: FAIL` and `round >= max_rounds` → escalation, `CLASS: missing_context`, recommendation: "split task / refine acceptance criteria / refine evidence contract." Task status: `escalated`.
- `VERDICT: BLOCKED` → escalation immediately, task status: `blocked`.

### 4. Arbiter (built-in `reviewer`)

When Crucible returns PASS, dispatch the **built-in `reviewer` agent** with the persona name "Arbiter" in the prompt for log clarity. Pass it: `run_id`, `task_id`, `spec_path`, `log_path`, the worktree path, the task's `acceptance_criteria` and any `evidence_contract` for context.

The reviewer's five passes still run. Two task-loop-specific overlays:

- **Pass 0 (divergence):** does the diff stay inside `files_touched`? Any file outside → `[scope-drift]` finding.
- **Pass 2 (edge cases) overlay:** does the diff strip or weaken any prior evidence contract field referenced by other tasks? Any stripping → `[evidence-regression]` blocking finding.

On return:
- `VERDICT: ship` → **check adversary triggers (step 4a)**. If any fire, dispatch adversary before merging. Otherwise merge worktree into merge_base, set task `status: done`, append merge SHA to log.
- `VERDICT: ship with fixes` → if blocking findings exist and `round < max_rounds`, treat as Crucible FAIL — Arya round+1 with reviewer findings as `prior_feedback`. If blocking findings exist and `round >= max_rounds`, escalate with `CLASS: risk`, recommendation: "split task / revise spec / accept manual fix." If only non-blocking, **check adversary triggers (step 4a)**; otherwise set `status: done` and append findings as `followups:` in the queue.
- `VERDICT: do not ship` → escalation, `CLASS: risk`, task status: `blocked`. (Adversary is not dispatched on broken code — fix the basics first.)

### 4a. Adversary (conditional, code target)

**Triggers (any of):**
- Diff size ≥ 50 changed lines (excluding generated files and lockfiles)
- Task touches a file in a risk-tagged path declared in `docs/ARCHITECTURE.md#risk-tagged-paths` (e.g., `src/auth/**`, `src/payments/**`, `migrations/**`, `src/middleware/external/**`)
- Task lists any `flagship_scenarios`
- The run was created in or escalated to a `high-risk` lane

If none fire, skip to Cleanup as if the task were done (post-Arbiter ship path).

**Dispatch.** Dispatch the local `adversary` agent (sonnet, target: `code`) with the standard contract. If the compound-engineering plugin is installed AND the user has opted in via run-config (`manifest.yaml#use_plugin_adversary: true`, default false), ALSO dispatch `compound-engineering:ce-adversarial-reviewer` in parallel via the Agent tool with the same context. Both write findings; merge them at return time (dedup by `file:line`; on severity disagreement, take the higher severity).

**Model-diversity guard.** Before dispatch, verify the adversary's model differs from the artifact's primary reviewer (Arbiter). If they match, log a configuration error to the task log and skip the adversary stage — do not silently launder blind spots. Surface to user as a queue-level escalation (`CLASS: misconfiguration`).

**Triage rubric (orchestrator applies, deterministic).** Walk each finding in order:

```
For each finding (severity, category, files-touched, recommended-action):

  1. severity == confirmed-real AND finding touches { auth | payment | PII | migration | data-loss }
     → ESCALATE  (class: adversary_finding, subclass: high_risk_domain)

  2. severity == confirmed-real AND task lists flagship_scenarios
     → ESCALATE  (class: adversary_finding, subclass: flagship_touched)

  3. severity == confirmed-real AND fix is within task.files_touched
     → ACCEPT    (Arya round+1 with this finding in prior_feedback)

  4. severity == confirmed-real AND fix requires widening files_touched
     → ESCALATE  (class: scope — same as any other scope decision)

  5. severity == plausible AND (risk_tagged_path OR flagship_scenario)
     → ESCALATE  (class: adversary_finding, subclass: plausible_high_risk — human reads the argument)

  6. severity == plausible AND not high-risk
     → REJECT-WITH-DOC  (logged to docs/decisions/, ship anyway with rationale)

  7. severity == speculative
     → REJECT-WITH-DOC  (logged to queue task.followups[], ship)
```

**Resolution per disposition:**

- **ACCEPT:** append finding(s) to `prior_feedback`, dispatch Arya at `round+1`. The next Arya round addresses both the original spec and the adversary's findings. After Arya returns DONE, run Crucible → Arbiter → adversary again (one extra round on the same task). Round cap (`max_rounds`) still applies; if the adversary keeps producing new accept-level findings past the cap, escalate with rationale.

- **REJECT-WITH-DOC:** write `docs/decisions/<date>-adversary-reject-<task_id>-<F#>.md` with: the finding (verbatim), the reason for rejection (one sentence), the cost-of-miss estimate, and which rule fired. For severity=speculative, instead append to `queue.yaml` task entry under `followups:` rather than creating a decisions file (avoid file proliferation for low-confidence items). Set task `status: done`, proceed to Cleanup.

- **ESCALATE:** append to `escalations.yaml.inbox` with `class: adversary_finding`, the full packet (severity, hypothesis, attack vector, reproduction if any, suggested mitigation, the rule that fired). Task `status: blocked`. Worktree preserved. Loop continues on other ready tasks. User resolves via `/sb-resolve`.

**All triage decisions are logged.** Append a "Round N — Adversary triage" block to `logs/<task-id>.md`:

```markdown
## Round <N> — Adversary triage — <ISO date>

**Findings received:** <n confirmed-real / n plausible / n speculative>

**Triage results:**
- F1 (confirmed-real, ReDoS in email regex) → ACCEPT (within files_touched; Arya round+1)
- F2 (plausible, race on session counter) → ESCALATE (rule 5 — flagship scenario FS04)
- F3 (speculative, cleanup of orphan worktree dirs) → REJECT-WITH-DOC (followup, low confidence)

**Resolution dispatched:** Arya round 3 / Escalation E007 / decisions/2026-05-29-adversary-reject-T042-F03.md
```

### 5. Cleanup

- If `done`: prune the worktree, write merge commit SHA to log, decrement in-flight counter, recompute ready frontier.
- If `blocked` / `escalated`: preserve the worktree for human inspection. Append worktree path to the escalation packet.

## Escalation format (file-based, async)

Each ESCALATE return becomes an entry in `docs/runs/<run-id>/escalations.yaml`:

```yaml
inbox:
  - id: E001
    task: T042
    raised_by: arya | crucible | arbiter | adversary
    round: 2
    class: architectural | strategic | risk | scope | missing_context | evidence_violation | adversary_finding | misconfiguration
    subclass: <optional, used by adversary_finding: high_risk_domain | flagship_touched | plausible_high_risk>
    question: "<one sentence>"
    context: "<2 sentences with file/section refs>"
    options:
      - A: "<option>"
      - B: "<option>"
    recommendation: "<pick + rationale>"
    raised_at: <ISO timestamp>
    worktree: .worktrees/<run-id>/<task-id>/   # preserved
    blocks: [T043, T051]                        # downstream tasks waiting on this
    # adversary_finding-specific fields (omitted for other classes):
    severity: confirmed-real | plausible | speculative
    attack_vector: "<concrete steps the adversary constructed>"
    reproduction: "<shell command or test snippet, or null>"
    suggested_mitigation: "<one sentence>"
    triage_rule_fired: <1..7>
```

The loop never waits inline. It writes the packet, moves on to other ready tasks, and prints to the user: "Task T042 escalated (E001) — loop continuing on other work." The user runs `/sb-resolve <run-id>` when ready.

## Progress surfacing (during the run)

Print a single live summary line at each scheduling tick:

```
[run:auth-rewrite] T012 done · T015 reviewing · T018 testing-r2 · T021 ready · 3 blocked · 14 of 28 done
```

After every 5 tasks complete or on quiescence, print a fuller block:

```
─── /sb-loop tick @ <time> ────────────
done:        T001..T014 (14)
in flight:   T015 (arbiter), T018 (crucible r2), T020 (arya r1)
ready next:  T021, T022, T024
blocked:     T010 (E003), T017 (E004)
escalations: 2 pending (/sb-resolve auth-rewrite)
```

Do not surface per-task transcripts in the live stream — they're in `logs/`. The live view is the dashboard, the logs are the audit trail.

## Resumability

State lives entirely on disk. After a crash or interruption:

1. Re-running `/sb-loop <run-id>` (with or without `--resume`) is idempotent.
2. Tasks marked `in_progress` at startup get their heartbeat checked (manifest stores last-tick timestamp). If stale (>10 min), reset to `ready` and surface "Reset stale in-flight tasks: T###." The next round runs as fresh, reading the partial log.
3. `done`, `blocked`, `escalated` tasks are not re-run.
4. Worktrees are inspected: orphan worktrees (no matching `in_progress` task) get logged but not deleted — they may hold uncommitted work.

## Quiescence and exit

When no tasks can advance:

- **All `done`** → manifest `status: completed`. Surface final summary. Suggest `/sb-review` over the merged diff, then `/sb-eval` once metrics window allows.
- **Some `blocked` / `escalated`** → manifest `status: drained`. Surface: "Drained — N tasks blocked. Run `/sb-resolve <run-id>` to unblock and re-run `/sb-loop <run-id>`."
- **Mixed** → same as drained, with the done-vs-blocked split surfaced.

## Decision routing during the loop

The orchestrator (you) makes **no decisions** about task content. You schedule, dispatch, write state, and surface. Decisions belong to:

- **Personas** for tactical and technical, logged in `logs/<task-id>.md`.
- **The user via `/sb-resolve`** for architectural, strategic, risk, scope, missing_context, and evidence_violation classes.

If you find yourself wanting to "just decide" something — re-route a dependency, widen a `files_touched`, retry a third time — stop. Escalate.

## Rules

- **No SURFACE checkpoints in the inner loop.** That's the whole point. Use escalation queue.
- **Never silently widen `files_touched`.** A persona requesting it returns ESCALATE with `CLASS: scope`.
- **Never bypass the Evidence Contract gate.** A flagship-touching task with `evidence_violation` does not advance even if all correctness tests pass.
- **Never modify completed tasks.** A done task is a commitment. If a later task reveals it was wrong, that's a new task or an escalation.
- **Always persist before dispatching.** State changes (status updates, log appends, escalation writes) happen before the next agent call, so a crash mid-tick is recoverable.
- **Worktrees are sacred.** Do not write to the main checkout from inside the loop. Personas write to worktrees; orchestrator merges to main only on `done`.
