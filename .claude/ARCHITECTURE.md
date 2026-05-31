# Architecture (workflow contract)

Eleven commands, eleven agents, two skills, one optional hook. The **default execution mode is the autonomous loop** (`/sb-spec` → `/sb-loop` ↔ `/sb-resolve`), preceded by domain research (`/sb-research`) when needed, gated by an independent inception audit, and adversarially reviewed at both spec time and code time. The per-feature pipeline (`/sb-plan` → `/sb-work`) is the manual lane for one-off changes that don't justify a full run. `/sb-view` renders research artifacts to a self-contained HTML site at `.local/research-views/` for human consumption.

Read in 5 minutes.

## Lifecycle stages

```
RESEARCH (optional, upstream)
    /sb-research <topic-slug>   ──→  docs/research/<topic>/
                                     ├─ INSIGHTS.md   (load-bearing, read by intake)
                                     ├─ BRIEF.md      (on-demand)
                                     ├─ SOURCES.md
                                     └─ sources/*
                                     + appends to docs/research/INDEX.md

VISION (gated by independent audit + advocate)
    /sb-init   ──→ intake-author (opus, reads research INSIGHTS)
                ──→ STRATEGY.md  ARCHITECTURE.md  ROADMAP.md  (draft)
                ──→ inception-auditor (sonnet, independent of intake)
                ──→ devil-advocate (sonnet, target: strategy)
                ──→ SURFACE all three; user approves or revises

ONE-SHOT THE PROJECT (default execution)
    /sb-spec <run-id>   ──→ planner per feature in spec mode (opus)
                         ──→ elicits metrics + flagship_scenarios + evidence_contracts
                         ──→ docs/runs/<run-id>/{spec.md, queue.yaml, manifest.yaml, escalations.yaml, logs/}
                         ──→ SURFACE for single approval

    /sb-loop <run-id>   ──→ scheduler; per task in its own worktree:
                              Arya (opus) → Crucible (opus) → Arbiter (built-in reviewer)
                              flagship tasks: Evidence Contract gate (Tier 0)
                              parallel where files_touched and components are disjoint
                              escalations are async — written to escalations.yaml

    /sb-resolve <run-id> ──→ batch-resolve escalations; decisions → docs/decisions/
                          ──→ run /sb-loop again to resume

AFTER SHIP
    /sb-eval <feature>  ──→ measure named metrics

ANYTIME
    /sb-plan <feature> → /sb-work   (manual lane for one-off changes)
    /sb-review                       (code review on the current diff)
    /sb-sanity                       (drift check across docs/code; optional handoff snapshot)
```

## Roles

| Role | Model | Job | Hard rules |
|---|---|---|---|
| **Orchestrator** | (the main session) | Dispatch agents. Never edit. Never audit. SURFACE outputs between stages (manual lane) or write file-based escalations (autonomous loop). | Lives in `.claude/commands/`. |
| **Intake Author** | opus | Conduct four-lens product inception. Produces STRATEGY/ARCHITECTURE/ROADMAP. Reads research INSIGHTS if present. | One question at a time. Honor existing context. |
| **Inception Auditor** | **sonnet** | Critique STRATEGY/ARCHITECTURE/ROADMAP for coherence, completeness, groundedness. **Independent model** from the intake author. | Read-only. Fresh context. Returns findings, never edits. |
| **Planner** | opus | Two modes: `plan` (per feature, used by /sb-plan) and `spec` (project-wide, used by /sb-spec). Elicits metrics + flagship scenarios + evidence contracts. | Reads actual code. No silent scope expansion. |
| **Auditor (plan/requirements/drift)** | opus | Critique plan/requirements artifacts. Three surface modes plus Evidence Contract Traceability. | Read-only. No pre-digestion. |
| **Concerns Auditor** | opus | Walk product-fit concerns (scale, security, accessibility, reliability, observability) per /sb-plan stage 4. | Read-only. Skill-driven checklists. |
| **Devil's Advocate** | **sonnet** | Argue against shipping. Two targets: `plan` (per /sb-plan stage 5) and `strategy` (per /sb-init final audit). **Independent model** from the artifact's author. | Read-only. One round. State case, accept verdict, move on. |
| **Adversary** | **sonnet** | Generative — constructs failure scenarios to break the artifact. Two targets: `spec` (mandatory at /sb-spec for flagship features; catches missing failure modes upstream) and `code` (conditional at /sb-loop after Arbiter on triggered tasks; catches implementation-specific exploits no spec could predict). | Read-only (Bash allowed for probing the worktree, never writing source). Reproductions promote findings to `confirmed-real`. |
| **Research Renderer** | sonnet | Reads research artifacts under `docs/research/` and emits a self-contained HTML site under `.local/research-views/`. Two modes — `dashboard` (cross-topic index) and `topic` (single topic + drill-downs + per-source pages). Faithful presentation; never paraphrases. | Output is gitignored. No network, no JS, no CDN — pages work offline via `file://`. |
| **Reviewer (Arbiter)** | opus | Critique code after implementation. Five passes plus scope-drift and evidence-regression overlays in `/sb-loop`. | Frame-switching mandatory. Quote file+line. |
| **Arya** | opus | Developer persona inside `/sb-loop`. Implements one task per dispatch. | Full tools inside the task's worktree. Logs evidence emissions when task touches a flagship scenario. |
| **Crucible** | opus | Tester persona inside `/sb-loop`. Three-tier testing: Evidence Contract gate → acceptance → edge/regression. | Writes test files only. Failure on Tier 0 short-circuits Tier 1 and 2. |
| **Human** | (you) | Resolve escalations, set direction. | Inline in `/sb-plan`; async via `/sb-resolve` in `/sb-loop`. |

### Why model diversity for audits

The intake author and the planner run on **opus**. The inception auditor and devil's advocate run on **sonnet**. This isn't a cost choice — it's a blind-spot choice.

When the auditor runs on the same model as the author, systematic biases of that model survive into the audit. Different model = different distribution of mistakes and catches. The sonnet auditor finds things the opus author missed for opus-shaped reasons; the opus reviewer in `/sb-loop` finds things sonnet might have missed for sonnet-shaped reasons (but Crucible's mechanical Evidence + Acceptance gate runs before the reviewer, so most discipline-of-the-loop is enforced before model bias matters).

If you swap model assignments later (e.g., to keep cost down), preserve the **diversity principle**: never audit an artifact with the same model that produced it. The orchestrator commands include a guard against accidental matching.

## Dispatch flow

```
Manual lane (/sb-plan, /sb-work, /sb-review):
  User invokes /command
       ↓
  Orchestrator (command)
       ↓
  Agent (implementor or auditor)
       ↓
       ├── DONE → orchestrator → SURFACE to user → next step
       └── ESCALATE → orchestrator → Human → decision logged → resume agent

Autonomous loop (/sb-loop):
  User invokes /sb-loop <run-id>
       ↓
  Orchestrator (scheduler)
       ↓
  Per task (parallel up to max_concurrent):
    Arya → Crucible → Arbiter ── done → merge worktree
       │       │         │
       │       ↓ FAIL    ↓ REJECT
       │     Arya round+1 (max_rounds)
       │
       ↓ ESCALATE
    write to docs/runs/<run-id>/escalations.yaml
    mark task blocked
    continue scheduling other tasks

  Quiescence → user runs /sb-resolve <run-id> → user runs /sb-loop <run-id>
```

One-hop only. Agents don't dispatch agents. Orchestrators never do agent work themselves.

## Decision routing

Agents classify decisions by blast radius:

- **Tactical** — naming, imports, formatting, local refactors → decide silently
- **Technical** — library options, implementation pattern, cache shape, test strategy → decide and log under the plan's or task's log
- **Architectural** — component boundary, public interface, dependency direction, data model, new external service → escalate
- **Strategic / risk** — scope, product direction, user promise, irreversible or high-blast-radius action → escalate
- **Scope** (autonomous loop only) — task wants to widen `files_touched` beyond its declared set → escalate

Use this rubric in order:

1. **Could this change the product promise, user scope, roadmap priority, red line, or success metric?** Strategic / risk.
2. **Could this be hard to reverse, expose data, affect auth/payments/PII/schema/migrations, or change behavior outside the current feature?** Strategic / risk.
3. **Could another feature reasonably depend on this boundary, interface, data model, dependency direction, or external service choice?** Architectural.
4. **Does this widen the declared `files_touched` for an autonomous-loop task?** Scope.
5. **Is this a meaningful implementation choice inside approved boundaries, with reversible consequences?** Technical.
6. **Is this only local naming, organization, formatting, or glue code with no behavioral or interface impact?** Tactical.

Every escalation includes a recommendation with rationale so you can disagree with the *reasoning*, not just pick a letter. Human-resolved decisions log to `docs/decisions/` and future agents read them as precedent.

**Channel:**
- Manual lane: inline SURFACE in the orchestrator → human decides immediately.
- Autonomous loop: written to `docs/runs/<run-id>/escalations.yaml`, batched, resolved via `/sb-resolve`.

## Adversary triage rubric

When the Adversary (code target) returns findings inside `/sb-loop`, the orchestrator applies a deterministic rubric to route each finding into one of three buckets. **The orchestrator does the triage; the user is involved only when the rubric routes to escalate.**

```
For each finding (severity, files-touched, category):

  1. confirmed-real AND touches { auth | payment | PII | migration | data-loss }
     → ESCALATE   (adversary_finding / high_risk_domain)

  2. confirmed-real AND task lists flagship_scenarios
     → ESCALATE   (adversary_finding / flagship_touched)

  3. confirmed-real AND fix is within task.files_touched
     → ACCEPT     (Arya round+1 with finding as prior_feedback; logged)

  4. confirmed-real AND fix needs to widen files_touched
     → ESCALATE   (scope class — same as any other scope decision)

  5. plausible AND (risk_tagged_path OR flagship_scenario)
     → ESCALATE   (adversary_finding / plausible_high_risk)

  6. plausible AND not high-risk
     → REJECT-WITH-DOC   (cost-of-miss logged to docs/decisions/, ship)

  7. speculative
     → REJECT-WITH-DOC   (appended to task.followups, ship)
```

**All triage decisions are logged.** ACCEPT writes to the task's `logs/<task-id>.md`. REJECT-WITH-DOC writes to `docs/decisions/<date>-adversary-reject-<task>-<finding>.md` (severity ≥ plausible) or to `queue.yaml#task.followups[]` (severity = speculative). ESCALATE writes to `escalations.yaml.inbox` and the user resolves via `/sb-resolve`.

**At spec time** (`/sb-spec`, target: spec), the triage is lighter because the user is present at SURFACE. Findings are walked one at a time with a `revise spec | defer to ## Assumptions` choice — no async escalation channel because the human is already in the loop.

**Why the orchestrator (not the adversary) decides.** The adversary proposes an action with each finding, but the orchestrator owns the rubric. Two reasons: (1) the rubric is deterministic and project-specific (knows your `risk-tagged-paths` and `flagship_scenarios`), so it shouldn't be re-derived inside the adversary's model context; (2) the orchestrator is also where `files_touched` scope and decisions-log precedent live. Splitting "I found this" (adversary) from "we should do X about it" (orchestrator) keeps each agent's job tight.

## Evidence Contract Coverage

The autonomous loop has a third class of gate alongside acceptance criteria and metrics: **Evidence Contracts** on flagship scenarios.

A feature designates 0-3 **flagship scenarios** — scenarios where production diagnosability is load-bearing. For each, the planner writes an `evidence_contract`: the fields that must be emitted with the right shape, in the right channel, so an investigator can answer the decisive diagnostic question without reading code.

**Required schema per field:**
- `field` — name
- `decisive_for` — which diagnostic question this field answers (forces "why" thinking)
- `must_appear_in` — channels from the canon: `log`, `metric`, `trace_span`, `response_body`, `response_header`, `audit_record`
- Optional: `allowed_values`, `pii_constraint`

**Three enforcement points:**

1. **At `/sb-spec` time** — planner proposes scenarios + contracts per feature; user approves via SURFACE checkpoint. Auditor (plan mode) builds an Evidence Contract Traceability table; a missing-field or off-canon-channel finding is blocking.
2. **At `/sb-loop` time** — Crucible runs **Tier 0 (Evidence)** before Tier 1 (Acceptance) and Tier 2 (Edge/Regression). A flagship-touching task whose evidence is missing or wrong-shape FAILs with `failure_class: evidence_violation` and Arya gets another round to add the emission. The acceptance tests don't even run until visibility passes.
3. **At Arbiter (built-in `reviewer`) time** — Pass 2 (edge cases) overlay checks the diff for evidence-field regression: did this change accidentally strip or rename a field declared in any contract?

The deliberate ordering: **a system that "works" but is undiagnosable in prod is a worse outcome than one that visibly doesn't work.** The second one gets fixed.

## The autonomous loop in detail

```
/sb-init <description or PDF>          [DEEP MODE: 30-60 min conversation]
  └── four lenses: business, user, product, technical
       └── produces STRATEGY.md, ARCHITECTURE.md, ROADMAP.md

──────────────────────────────────────────────────────────────────

/sb-spec <run-id> [scope hint]         [pre-execution, one approval]
  ├── Step 1: scope confirmation                                SURFACE
  ├── Step 2: planner per feature (spec mode)
  │     ├── elicit metrics (per feature)                        SURFACE
  │     └── elicit flagship_scenarios + evidence_contracts      SURFACE
  ├── Step 3: build queue.yaml (atomicity + disjoint-files rule)
  ├── Step 4: write manifest.yaml + escalations.yaml (empty)
  └── Step 5: full picture approval                             SURFACE

/sb-loop <run-id>                      [autonomous execution]
  while not quiescent:
    ready = todo tasks with all deps done AND files_touched disjoint with in-flight
    for each picked task (up to max_concurrent, in own worktree):
      Arya round 1
        ESCALATE → escalations.yaml.inbox + status:blocked + continue
        DONE → Crucible round 1
          Tier 0: Evidence Contract gate (if flagship)
            FAIL → Arya round+1 with evidence_violation feedback
          Tier 1: acceptance
          Tier 2: edge + regression
          PASS → Arbiter (built-in reviewer)
            Pass 0: divergence (scope check vs files_touched)
            Pass 1-4: correctness, edge cases, security, simplicity
            Pass 2 overlay: evidence regression
            ship → merge worktree, status:done
            ship-with-fixes → Arya round+1 OR done + followups
            do-not-ship → escalation
          FAIL → Arya round+1 (max_rounds cap)
          BLOCKED → escalation
    update queue.yaml, persist state, surface live tick

/sb-resolve <run-id>                   [batch escalation resolution]
  for each escalation in inbox:
    SURFACE packet verbatim
    user picks A/B/custom/defer/abandon/inspect/re-dispatch
    log to docs/decisions/<date>-<slug>.md
    queue.yaml: status blocked → ready with decision as prior_feedback

/sb-loop <run-id>                      [resume after resolve]
  picks up newly-ready tasks
```

## Manual lane (still supported)

For one-off changes that don't justify a full run:

```
/sb-plan <feature>                     [pre-execution pipeline]
  ├── Stage 1: requirements audit       SURFACE
  ├── Stage 2: planner (plan mode)      SURFACE
  ├── Stage 3: plan audit               SURFACE
  ├── Stage 4: concerns audit           SURFACE  (skipped if --quick)
  ├── Stage 5: devil's-advocate         SURFACE  (skipped if --quick)
  └── Stage 6: user approval

/sb-work                               [execute the approved plan]
/sb-review                             [4-pass code review of the diff]
/sb-eval <feature>                     [post-ship measurement]
```

**When to use which:**
- **Autonomous loop:** multi-feature work, parallel-friendly scope, willing to do upfront spec, want to step away.
- **Manual lane:** one feature, prefer inline checkpoints, exploratory or contested scope where upfront spec would be premature.

The two modes share planner/auditor/reviewer/concerns-auditor/devil-advocate/intake-author/decisions log/STRATEGY/ARCHITECTURE/ROADMAP. Only the orchestration differs.

## Namespace: `sb-` prefix

All custom commands use `sb-` ("super-builder") to avoid collisions with Claude Code's built-ins (`/init`, `/plan`, `/clear`, etc.). When a built-in does part of what we need, our command **composes** with it rather than replacing it. The clearest case is `/sb-init`, which invokes built-in `/init` first to scaffold `CLAUDE.md`, then layers the deep intake.

## Roadmap-driven parallelism

The ROADMAP.md produced by `/sb-init` annotates features with which architecture components they touch. `/sb-spec` reads this and builds a task queue where two tasks may run concurrently iff their `files_touched` sets are disjoint AND their components are independent. The scheduler in `/sb-loop` enforces this at dispatch time, putting each task in its own git worktree (`.worktrees/<run-id>/<task-id>/`).

This is the practical mechanism for the "parallelize feature development" goal — it falls out of having an architecture that defines clean component boundaries.

## Why this is small on purpose

A new engineer joining your team should read this file in 5 minutes and understand the workflow. The autonomous loop adds three commands and two agents over the original six/six template — every addition earns its place by removing a recurring failure mode (silent improvisation, slow per-feature iteration, undiagnosable prod failure).

When the value of a piece can be delivered by extending an existing piece rather than adding a new one, extend. We reuse the existing planner (added `spec` mode), the existing auditor (added Evidence Contract Traceability), and the existing reviewer (framed as "Arbiter" in the loop). The only genuinely new agents are Arya and Crucible because no pre-existing role mapped to "implementor inside a per-task worktree" or "tester whose failure short-circuits on missing diagnostic signal."
