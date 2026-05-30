---
description: Produce a full project spec and atomic task queue under a named run. The single approval gate before /sb-loop runs autonomously. Reads STRATEGY/ARCHITECTURE/ROADMAP; writes docs/runs/<run-id>/{spec.md, queue.yaml, manifest.yaml}.
argument-hint: "<run-id slug> [optional: scope hint, e.g., 'Horizon 1' or path to a feature subset]"
role: orchestrator
---

# Spec (autonomous-loop entry point)

You are the **orchestrator**. `/sb-spec` is the one-time, upfront planning gate for an autonomous run. After approval, `/sb-loop <run-id>` executes the queue without further SURFACE checkpoints — only escalations.

Run id: **$ARGUMENTS** (parse the first token as `<run-id>`; the rest is an optional scope hint)

## Why this exists

`/sb-plan` plans one feature at a time with five SURFACE checkpoints — appropriate when the human is in the inner loop. `/sb-spec` plans an entire run upfront so the inner loop can run autonomously. You approve once; the loop runs many times.

## Pre-flight

1. Parse `<run-id>` from `$ARGUMENTS`. Validate: lowercase kebab-case, 3-40 chars. If invalid or missing, ask the user for a slug (e.g., `auth-rewrite`, `dashboard-v2`).
2. Check `docs/runs/<run-id>/` exists. If yes, surface: "Run exists. Overwrite, augment, or pick a new slug?" Augmenting means adding new tasks to an existing queue without resetting completed ones.
3. Verify upstream artifacts exist:
   - `docs/STRATEGY.md` — required
   - `docs/ARCHITECTURE.md` — required (the queue's parallelism depends on component boundaries)
   - `docs/ROADMAP.md` — required (the source of features to expand into tasks)
   - If any are missing, stop and direct the user to `/sb-init` first.

## Procedure

### Step 1: Scope confirmation

Read ROADMAP.md. Surface to the user:

```
Run id: <run-id>
Roadmap horizons available:
- Horizon 1: <feature list>
- Horizon 2: <feature list>
- ...

Which features are in scope for this run? (Default: Horizon 1.)
```

User picks. Record their choice in `manifest.yaml`.

### Step 2: Dispatch planner per feature, build the spec

For each feature in scope, dispatch the existing `planner` agent in **spec-mode** — pass these anchors:

```
Mode: spec
Run id: <run-id>
Feature: <name from ROADMAP>
Target spec section: docs/runs/<run-id>/spec.md#F<NN>
Read: docs/STRATEGY.md, docs/ARCHITECTURE.md#components, docs/ROADMAP.md#F<NN>, relevant existing code
Produce: contract + acceptance criteria + flagship_scenarios with evidence_contracts + atomic task breakdown. No code yet.
```

The planner produces a per-feature spec section. You concatenate them into `docs/runs/<run-id>/spec.md`. Anchors:
- `#F<NN>` for each feature
- `#FS<NN>` for each flagship scenario inside a feature
- `#T<NNN>` for each task

**Two SURFACE checkpoints fire per feature during spec elicitation:**

1. **Metric elicitation** (existing planner behavior) — SURFACE the planner's metric questions; user answers; you relay back.
2. **Flagship scenario elicitation** (new) — SURFACE the planner's flagship-scenario proposal: which 1-3 scenarios per feature qualify as "diagnosability is load-bearing" and the proposed `evidence_contract` for each. User accepts, modifies, or skips per scenario.

These are the only SURFACE checkpoints inside `/sb-spec` aside from the final approval. After approval, no further checkpoints fire inside the loop.

### Step 2a: Flagship scenarios and Evidence Contracts (the new gate)

Each feature designates **0-3 flagship scenarios** — scenarios where production diagnosability is load-bearing. Rule of thumb: anything a paying customer would page about, anything in a regulated path (auth/payment/PII), anything where the failure mode is "wrong answer" not "crash."

For each flagship scenario, the planner proposes an `evidence_contract` — the list of fields that must be emitted with the right shape, in the right channel, so an investigator can answer the decisive diagnostic question without reading the code.

Required schema per field:

```yaml
evidence_contract:
  - field: <name>
    decisive_for: "<which diagnostic question this field answers>"
    must_appear_in: [log | metric | trace_span | response_body | response_header | audit_record]
    allowed_values: [<optional enum>]
    pii_constraint: "<optional: e.g. 'hashed, never raw email'>"
```

The `decisive_for` is the load-bearing field. It forces the planner to explain *which question this field answers*, which kills the "log everything just in case" failure mode.

**Channels canon:** `log`, `metric`, `trace_span`, `response_body`, `response_header`, `audit_record`. Anything outside this canon is rejected — propose an addition via escalation if your codebase has a genuinely different channel.

**Failure mode for downstream loop:** when Crucible runs a flagship-touching task, the Evidence Contract gate runs *before* correctness tests. If any field is missing or wrong-shape, the task FAILs with `failure_class: evidence_violation` and Arya retries — same retry semantics as any other Crucible FAIL.

### Step 2b: Spec-time adversarial review (the upstream catch)

For each feature whose flagship_scenarios is non-empty, dispatch the `adversary` agent with `Target: spec`. **Mandatory** for flagship features — `--skip-adversarial-spec` is the explicit override and must be confirmed by the user with a written reason logged to `docs/decisions/<date>-spec-adversary-skip.md`.

Why this stage exists: the spec is where failure modes get anticipated for the autonomous loop. A missed failure mode in the spec multiplies across every task implementing that feature — one spec gap, N implementation gaps. Catching it once at spec time prevents N catches at code time. The model-diversity rule applies: adversary runs on sonnet, planner runs on opus.

```
Target: spec
Run id: <run-id>
Feature id: <F##>
Spec path: docs/runs/<run-id>/spec.md
Feature anchor: #F<NN> (and any #FS<NN> flagship scenarios for this feature)
Research grounding: docs/research/<topic>/INSIGHTS.md for any relevant topics
```

When adversary returns, **SURFACE findings inline** — you're already at a SURFACE checkpoint in `/sb-spec`; this isn't async territory. Walk findings with the user one at a time using this triage rubric:

| Severity | Default action | When to override |
|---|---|---|
| `confirmed-real` | **Revise spec.** Re-dispatch planner for this feature with the finding as additional context. | Override to `defer` only if the user can articulate why the gap doesn't matter for Horizon 1 — log to `## Assumptions` in the spec section. |
| `plausible` | **User decides.** Adversary argued but couldn't prove. Two paths: address now (re-dispatch planner) or defer with rationale logged to `## Assumptions`. | — |
| `speculative` | **Defer.** Log to `## Assumptions` as "considered, low-confidence, not blocking" with the adversary's hypothesis quoted. | Promote to "address now" if the user has independent evidence the concern is real. |

**The triage rationale must be logged.** Either the spec is revised (the change *is* the log), or the deferral lands in `## Assumptions` with a one-line rationale quoting the adversary's hypothesis. Silent dismissal is not an option — the audit trail makes future re-runs honest.

### Step 3: Build the queue

After all feature specs are written, build `docs/runs/<run-id>/queue.yaml`. Format:

```yaml
run_id: <run-id>
created_at: <ISO date>
spec_path: ./spec.md
manifest_path: ./manifest.yaml
schema_version: 1

# Defaults applied to every task unless overridden
defaults:
  max_rounds: 3            # dev↔tester loop cap before forced escalation
  worktree: true           # each task runs in its own git worktree
  status: todo

features:
  - id: F01
    title: "Login flow"
    component: auth
    flagship_scenarios:    # 0-3 per feature; full evidence_contract lives at spec.md#FS<NN>
      - FS01               # "Failed login due to wrong password"
      - FS02               # "Account locked after N failed attempts"

tasks:
  - id: T001
    feature: F01
    title: "Create user model with email uniqueness"
    component: auth        # from ARCHITECTURE.md#components — drives parallelism
    depends_on: []         # task IDs that must be `done` first
    files_touched:         # used for conflict detection at scheduling time
      - src/models/user.ts
    flagship_scenarios: [] # which FS### this task implements/touches; empty = no Evidence gate
    acceptance_criteria:   # short form; full text lives at spec.md#T001
      - "User created with valid email returns user object"
      - "Duplicate email returns validation error 409"
      - "Empty email returns validation error 400"
    status: todo           # todo | ready | in_progress | testing | reviewing | done | blocked | escalated | abandoned
    notes: ""              # human or persona annotations; non-load-bearing

  - id: T012
    feature: F01
    title: "Wire failure_reason and request_id into login response and log"
    component: auth
    depends_on: [T001, T009]
    files_touched:
      - src/auth/login.ts
      - src/middleware/correlation.ts
    flagship_scenarios: [FS01]   # ← triggers Crucible's Tier 0 Evidence Contract gate
    acceptance_criteria:
      - "Successful login returns 200 with user payload"
      - "Wrong password returns 401"
    status: todo
    notes: ""
```

**Task atomicity rule:** each task is one Arya → Crucible → Arbiter cycle. If a feature is too large for one cycle, the planner splits it into multiple T-tasks with `depends_on` edges.

**Disjoint-files rule:** two tasks may run concurrently only if their `files_touched` sets are disjoint. The planner enforces this at queue-build time by splitting tasks or adding dependencies; you (the orchestrator) verify it before the user approves.

### Step 4: Manifest

Write `docs/runs/<run-id>/manifest.yaml`:

```yaml
run_id: <run-id>
created_at: <ISO date>
created_by: <git config user.email or "unknown">
scope:
  source: docs/ROADMAP.md
  features: [F01, F02, ...]
spec_path: ./spec.md
queue_path: ./queue.yaml
escalations_path: ./escalations.yaml
logs_dir: ./logs/
merge_base: <git rev-parse HEAD at spec time>
status: pending_approval   # pending_approval | running | paused | drained | completed
```

Also create empty `docs/runs/<run-id>/escalations.yaml`:

```yaml
run_id: <run-id>
schema_version: 1
inbox: []
resolved: []
```

And create `docs/runs/<run-id>/logs/` directory (empty).

### Step 5: SURFACE for approval (the only gate)

Surface the full picture:

```
Run: <run-id>
Features in scope: <list>
Tasks: <count> total / <count> with no deps (initial ready frontier)
Components touched: <list from queue>
Parallelism estimate: <max concurrent tasks at any point given the dep graph>

Spec: docs/runs/<run-id>/spec.md
Queue: docs/runs/<run-id>/queue.yaml
```

Ask one question: **proceed to /sb-loop, revise, or stop?**

- **proceed** → tell the user to run `/sb-loop <run-id>`. Do not auto-trigger.
- **revise** → which feature/task? Re-dispatch planner with the user's revision.
- **stop** → leave files in place; user resumes later via re-running `/sb-spec <run-id>` with augment mode.

## What stays as SURFACE (after approval, none)

After approval, `/sb-loop` runs with **no further SURFACE checkpoints in the inner loop**. The only interruption is escalation — which is async (file-based) by default. The human comes back, runs `/sb-resolve <run-id>`, and resumes the loop.

This is the deliberate tradeoff: more upfront ceremony, no per-task ceremony.

## Decision routing during spec

Same rubric as `/sb-plan`. Tactical → silent, technical → log under the planner's `## Decision log` in the spec, architectural/strategic → escalate to the user during the spec conversation. After approval the escalation channel is file-based; during spec it's still SURFACE because the user is here.

## Rules

- One run id per `/sb-spec` invocation. Do not auto-create a second run.
- The spec is the single source of truth for the loop. Arya, Crucible, and Arbiter all read it. Don't bury decisions in side channels.
- If the planner can't satisfy the disjoint-files rule (genuinely entangled work), surface to the user — proceed serially, or revise the architecture so the entanglement goes away.
- Do not write any product code in `/sb-spec`. This is a planning stage only.

## Handling escalations from the planner

Standard pattern: surface the packet, wait for the user, log to `docs/decisions/<date>-<slug>.md`, resume the planner. The decisions log accumulates precedent the autonomous loop reads later.
