# Compound-starter

A Claude Code workflow template for **one-shotting a project** with an autonomous developer / tester / reviewer / adversary loop, gated by independent (different-model) audits and grounded in optional upstream domain research. Includes a manual lane for one-off changes.

**Ten commands, ten agents, two skills, one optional hook.** Read this file in 5 minutes and you'll know when to run what.

## The default flow

1. **(Optional)** `/sb-research <topic>` — domain research that grounds vision without flooding context.
2. `/sb-init` — product inception, then gated by an **inception audit (sonnet)** and a **devil's advocate against the strategy (sonnet)** — independent of the intake author (opus).
3. `/sb-spec <run-id>` — project-wide spec with metrics + flagship scenarios + evidence contracts. One approval gate.
4. `/sb-loop <run-id>` — autonomous execution. **Arya / Crucible / Arbiter** persona graph per task. Parallel where deps allow. Async escalations.
5. `/sb-resolve <run-id>` — batch-handle escalations when they accumulate.
6. `/sb-eval <feature>` — post-ship measurement against named metrics.

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Vision               One-shot the project (DEFAULT)    After ship  │
│   ───────             ─────────────────────────────      ──────────  │
│                                                                      │
│   /sb-init             /sb-spec <run-id>                 /sb-eval    │
│        │                    │                                 ↑      │
│        ↓                    ↓ (approve once)                  │      │
│   STRATEGY.md          docs/runs/<run-id>/                    │      │
│   ARCHITECTURE.md      ├─ spec.md                             │      │
│   ROADMAP.md           ├─ queue.yaml                          │      │
│                        ├─ manifest.yaml                       │      │
│                        ├─ escalations.yaml                    │      │
│                        └─ logs/<task-id>.md                   │      │
│                             │                                 │      │
│                             ↓                                 │      │
│                        /sb-loop <run-id>                      │      │
│                             │   per task, in own worktree:    │      │
│                             │   Arya → Crucible → Arbiter    │      │
│                             │   parallel where deps allow     │      │
│                             ↓                                 │      │
│                        escalations? ─yes─→ /sb-resolve <run-id>      │
│                             │ no                  ↓                  │
│                             │                resume /sb-loop         │
│                             ↓                                 │      │
│                        ──── ship ────────────────────────────┘       │
│                                                                      │
│   Manual lane:    /sb-plan <feature> → /sb-work → /sb-review         │
│   Anytime:        /sb-sanity (drift), /sb-review (code review)       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## The persona graph (per task, inside `/sb-loop`)

```
┌─────────────────────────┐
│  Orchestrator picks T###│
│  Creates worktree       │
└────────────┬────────────┘
             ↓
   ┌─── Arya (round N) ───┐    developer persona; implements one task
   │   reads spec, code,   │    against acceptance criteria.
   │   prior round's log   │    full write access inside the worktree.
   └────────────┬──────────┘
                ↓
   ┌── Crucible (round N) ─────┐  tester persona. Three tiers, in order:
   │ Tier 0: Evidence Contract │  → can the investigator see the answer?
   │ Tier 1: Acceptance        │  → does the system give the right answer?
   │ Tier 2: Edge + regression │  → does it hold under stress and history?
   └─┬────┬─────────────┬──────┘
     │    │ FAIL        │ BLOCKED
     │    │  → Arya round N+1 (max_rounds cap, default 3)
     │    └──→ escalation
     ↓ PASS
   ┌── Arbiter (built-in reviewer) ──┐  five passes: divergence,
   │  correctness / edges / security /│  correctness, edges, security,
   │  simplicity / evidence-survival  │  simplicity. plus scope-drift and
   └─┬─────────────────┬──────────────┘  evidence-regression overlays.
     │ ship            │ ship-with-fixes / do-not-ship
     ↓                 ↓
  done                escalation OR Arya round N+1
  merge worktree
```

## The artifact chain (autonomous loop)

```
Assignment / PDF / idea
     ↓
/sb-init  →  STRATEGY.md + ARCHITECTURE.md + ROADMAP.md
     ↓
/sb-spec <run-id>  →  docs/runs/<run-id>/{spec.md, queue.yaml, manifest.yaml}
     ↓
/sb-loop <run-id>  →  shipped code (per task: log + tests + merge commit)
     │
     ├─→ escalations.yaml grows when agents hit load-bearing decisions
     ↓
/sb-resolve <run-id>  →  docs/decisions/<date>-<slug>.md  ·  tasks unblock
     ↓
/sb-loop <run-id>  →  resumes
     ↓
/sb-eval <feature>  →  docs/evals/<feature>-<window>.md
```

`docs/runs/<run-id>/` is the single source of truth for a run. Multiple runs coexist without colliding. State on disk = resumability for free.

## What each command does

**Research (optional, upstream of vision):**
- `/sb-research <topic-slug>` — thin orchestration over the built-in `deep-research` skill. Reads PDFs, repos, URLs, freeform questions; produces a 3-tier output (`INSIGHTS.md` ~500 words = always loaded by intake-author; `BRIEF.md` 1-2 pages = referenced on demand; `sources/*` = full notes for deep dives). Appends to `docs/research/INDEX.md`. Use when domain context (learning science, competitor landscape, motivation studies for an age group, etc.) should ground inception without flooding it.

**Vision (run once at project start, refresh occasionally):**
- `/sb-init` — four-lens intake (business, user, product, technical). 30-60 minutes of real conversation. Produces STRATEGY/ARCHITECTURE/ROADMAP. `--quick` for a 6-question fast path producing only STRATEGY. **After intake, two independent reviews run automatically** — `inception-auditor` (sonnet) for coherence/completeness/groundedness, and `devil-advocate` (sonnet, strategy target) for argument-against-the-bet. Different model from the intake author by design.

**Autonomous one-shot (default):**
- `/sb-spec <run-id>` — pre-execution spec stage. Reads ROADMAP, dispatches `planner` in spec-mode per feature, elicits **metrics** and **flagship scenarios with Evidence Contracts**, builds an atomic `queue.yaml`. One SURFACE approval gate before the loop runs.
- `/sb-loop <run-id>` — autonomous scheduler. Per task: Arya → Crucible → Arbiter, parallel where dependencies allow, async escalations. No SURFACE checkpoints in the inner loop. Resumable.
- `/sb-resolve <run-id>` — batch escalation handler. Walks pending escalations one at a time, logs decisions to `docs/decisions/`, unblocks tasks, prints "loop is unblocked."

**Manual lane (one-off changes):**
- `/sb-plan <feature>` — five-stage pipeline with inline SURFACE checkpoints. Lane is inferred from the feature description; override via `--tiny`, `--quick`, or `--high-risk`.
- `/sb-work` — execute the approved plan. Stops if reality diverges. Mandatory divergence summary.
- `/sb-review` — code review with five passes (divergence + correctness, edges, security, simplicity) in sequence.

**After ship:**
- `/sb-eval <feature>` — measure actual outcomes against named metrics. 30-day default; `--window 7d` for shipping-quality check.

**Anytime:**
- `/sb-sanity` — drift check by a fresh-context agent. STRATEGY/ARCHITECTURE/ROADMAP/plans/code/CLAUDE.md alignment. Add `--with-handoff` for an orientation snapshot.

## When to use which mode

| You want to | Use |
|---|---|
| Build many features from a known roadmap, mostly hands-off | **Autonomous loop** (`/sb-spec` → `/sb-loop`) |
| Bring up a single feature with inline checkpoints | **Manual lane** (`/sb-plan` → `/sb-work` → `/sb-review`) |
| Tiny/quick fix (typo, dep bump, renamed function) | Manual lane, `/sb-plan --tiny` exits immediately and tells you to just make the edit |
| Touch something risky (auth/payments/PII/migration) | Either mode — Evidence Contract Coverage and the `risk` decision class handle the same concerns differently |
| Review a code change you didn't make | `/sb-review` standalone |
| Confirm STRATEGY/ARCHITECTURE/ROADMAP haven't drifted | `/sb-sanity` |

The two modes share planner / auditor / reviewer / concerns-auditor / devil-advocate / intake-author and the STRATEGY/ARCHITECTURE/ROADMAP/decisions log. Only the orchestration differs.

## Evidence Contract Coverage

The autonomous loop has a third gate alongside acceptance criteria and metrics: **Evidence Contracts on flagship scenarios.**

Each feature designates 0-3 flagship scenarios — where production diagnosability is load-bearing (anything a customer would page about, anything in auth/payment/PII, anything where "wrong answer" matters more than "crash"). Each gets an `evidence_contract`: the fields that must be emitted with the right shape, in the right channel, so an investigator can answer the decisive diagnostic question without reading code.

```yaml
flagship_scenarios:
  - id: FS01
    name: "Failed login due to wrong password"
    evidence_contract:
      - field: failure_reason
        decisive_for: "telling bad_password from locked/unknown without reading code"
        must_appear_in: [log]
        allowed_values: [bad_password, locked, unknown_user, rate_limited]
      - field: user_id_hash
        decisive_for: "answering 'is this user affected' from a ticket"
        must_appear_in: [log]
        pii_constraint: "hashed, never raw email"
```

**Enforcement, three points:**

1. **Spec time** — planner proposes contracts per feature; auditor (plan mode) builds an Evidence Contract Traceability table; a missing field is a blocking finding.
2. **Loop time** — Crucible runs **Tier 0 (Evidence)** before Tier 1 (Acceptance). A flagship-touching task whose evidence is missing FAILs with `failure_class: evidence_violation` — correctness tests don't even run. Arya gets another round to add the emission.
3. **Review time** — Arbiter's Pass 2 overlay checks the diff for evidence-field regression: did this change accidentally strip or rename a field declared in any contract?

The deliberate ordering: a system that "works" but is undiagnosable in prod is a worse outcome than one that visibly doesn't work. The second one gets fixed.

## Install

### New project
1. On GitHub, click **"Use this template"** → "Create a new repository."
2. Clone locally. Open in Claude Code.
3. Run `/sb-init` (or `/sb-init --quick` for light intake).
4. Then `/sb-spec <run-id>` to scope a run and `/sb-loop <run-id>` to execute.

### Drop into existing project
1. Copy `.claude/`, `CLAUDE.md`, `settings.json.example`, `TEMPLATE_VERSION`, `CHANGELOG.md` into the repo.
2. If the repo already has a `CLAUDE.md`, rename ours to `CLAUDE.md.template` and merge selectively.
3. Run `/sb-init` — even on existing repos, the deep intake gives you STRATEGY/ARCHITECTURE/ROADMAP grounded in what already exists.

## The strong opinions

These are the design calls worth knowing about. Full rationale in `.claude/ARCHITECTURE.md`.

1. **Default to autonomy with one approval gate.** The autonomous loop runs after a single upfront approval at `/sb-spec`. The manual lane stays available for one-offs, but it's no longer the default.
2. **Roles are strict.** Orchestrators dispatch; implementors produce; auditors critique with read-only tools; humans decide. Orchestrators never edit artifacts.
3. **One-hop dispatch.** Agents don't dispatch agents. Orchestrator is the single source of truth for what happened.
4. **Audit independence.** Auditors see the artifact, not the author's reasoning. Fresh context. No pre-digestion.
5. **Escalations are async by default in the loop.** When `/sb-loop` hits a load-bearing decision, it writes to `escalations.yaml` and continues with other ready tasks. You handle them in batch via `/sb-resolve`, not 50 interruptions.
6. **Metrics must be instrumented.** Requirements audit blocks if no metrics are named. Plan/spec audit blocks if named metrics have no instrumentation step. Eval reads the instrumented values.
7. **Evidence Contract Coverage.** Flagship scenarios get evidence contracts. Crucible runs visibility tests *before* correctness tests — if the investigator can't see the answer, the test of whether the answer is right is meaningless.
8. **Roadmap-driven parallelism.** ROADMAP.md annotates features with which architecture components they touch. The scheduler in `/sb-loop` runs tasks concurrently iff their `files_touched` sets are disjoint AND their components are independent. Each task in its own git worktree.
9. **State lives on disk; loops resume.** A run's queue, logs, manifest, and escalations are all files under `docs/runs/<run-id>/`. Worktrees under `.worktrees/<run-id>/<task-id>/`. Crash recovery is automatic: re-running `/sb-loop <run-id>` picks up where it left off.
10. **Decision routing keeps momentum without hiding risk.** Tactical decided silently; technical decided and logged; architectural / strategic / risk / scope (loop-only) escalated.
11. **Decisions log is the system's memory of your judgment.** Human-resolved decisions go in `docs/decisions/`. Future agents read them as precedent. Repeated decisions promote to CLAUDE.md.
12. **Eval is the truth source.** Strategy is aspirational until measured. The 30-day eval catches premise problems that the build loop misses.
13. **Compose with built-ins, never replace.** `/sb-init` invokes Claude Code's built-in `/init` first to scaffold CLAUDE.md, then layers the deep intake. `/sb-research` is a thin wrapper over the built-in `deep-research` skill — it only enforces output structure. The Arbiter persona is the existing built-in `reviewer` — no new agent for that role.
14. **Model diversity for audits.** Intake-author and planner run on **opus**; inception-auditor and devil-advocate run on **sonnet**. This isn't a cost choice — different model = different distribution of mistakes. An audit done by the same model as the author silently launders the same blind spots. Never audit an artifact with the model that produced it.
15. **Research is tiered, not dumped.** `/sb-research` produces `INSIGHTS.md` (always loaded by intake, ~500 words), `BRIEF.md` (1-2 pages, on demand), and `sources/*` (full notes, on demand). Intake-author reads only the index + relevant INSIGHTS — the corpus stays on disk. This is what keeps domain grounding affordable in context-economy terms.
16. **Adversarial review at two altitudes.** Spec-time adversary (`/sb-spec`, mandatory for flagship features) catches missing failure modes before they multiply across N tasks. Code-time adversary (`/sb-loop`, conditional on triggers) catches implementation-specific exploits no spec can predict — ReDoS, library CVEs, timing attacks, races, AI-shaped hallucinated APIs, evidence-key typos. Either alone leaves known blind spots; both is the disciplined answer.
17. **Adversary findings get triaged by the orchestrator, not the agent.** Deterministic rubric: `confirmed-real` in high-risk domain or flagship-touched → escalate; `confirmed-real` within scope → accept (Arya round+1); `plausible` in risk path → escalate; `plausible` non-risk → reject-with-doc; `speculative` → reject-with-doc as followup. Every triage decision logged. The user is only interrupted when the rubric routes to escalate.

## Personas in the loop

| Persona | Agent file | Model | Role |
|---|---|---|---|
| **Arya** | `agents/arya.md` | opus | Developer. One task per dispatch. Full write tools inside the task's worktree. Logs evidence emissions when task touches a flagship scenario. |
| **Crucible** | `agents/crucible.md` | opus | Tester. Three-tier testing (Evidence → Acceptance → Edge/Regression). Writes test files only. Returns PASS/FAIL/BLOCKED with reproduction. |
| **Arbiter** | (built-in `reviewer`) | opus | Code-review gate. Five passes plus scope-drift and evidence-regression overlays. Same agent as `/sb-review` — different framing inside the loop. |
| **Adversary** | `agents/adversary.md` | **sonnet** | Generative breaker. Constructs failure scenarios to break the implementation rather than checking against known patterns. Conditional: only fires when triggers match (diff ≥50 / risk path / flagship scenario / high-risk lane). |

## Inception-stage agents (model-diverse from intake)

| Agent | Model | Role |
|---|---|---|
| `intake-author` | opus | Conducts the four-lens intake conversation. |
| `inception-auditor` | **sonnet** | Critiques the STRATEGY/ARCHITECTURE/ROADMAP draft for coherence / completeness / groundedness. Different model from intake-author — that's the point. |
| `devil-advocate` (strategy target) | **sonnet** | Argues against the bet itself, not the execution. Same model-diversity rationale. |
| `adversary` (spec target) | **sonnet** | Constructs failure modes the spec doesn't anticipate. Mandatory at `/sb-spec` for any feature with non-empty `flagship_scenarios`. Catches spec gaps that would multiply across N implementation tasks. |

## File layout

```
.claude/
├── ARCHITECTURE.md                ← workflow contract (read this second, 5 min)
├── commands/                      ← orchestrators
│   ├── sb-research.md             ← upstream domain research (NEW)
│   ├── sb-init.md                 ← intake + inception audit + strategy advocate
│   ├── sb-spec.md                 ← autonomous-loop entry point
│   ├── sb-loop.md                 ← scheduler (default execution mode)
│   ├── sb-resolve.md              ← batch escalation handler
│   ├── sb-plan.md                 ← manual lane: per-feature pipeline
│   ├── sb-work.md                 ← manual lane: execute plan
│   ├── sb-review.md               ← code review (shared)
│   ├── sb-eval.md                 ← post-ship measurement
│   └── sb-sanity.md               ← drift check / handoff snapshot
├── agents/                        ← implementors + auditors
│   ├── intake-author.md           (opus) reads research INSIGHTS if present
│   ├── inception-auditor.md       (sonnet) audits intake's output  (NEW)
│   ├── planner.md                 (opus) plan mode + spec mode
│   ├── auditor.md                 (opus) requirements / plan / drift modes + evidence traceability
│   ├── concerns-auditor.md        (opus)
│   ├── devil-advocate.md          (sonnet) target: plan or strategy
│   ├── reviewer.md                (opus) also the Arbiter persona inside /sb-loop
│   ├── arya.md                    (opus) developer persona
│   ├── crucible.md                (opus) tester persona, 3-tier
│   └── adversary.md               (sonnet) generative breaker, two targets: spec, code  (NEW)
└── skills/
    ├── product-concerns/SKILL.md   (checklists for the concerns-auditor)
    └── doc-freshness/SKILL.md      (proposes doc updates at eval time)

CLAUDE.md                         ← project context (fill in TODOs)
README.md                         ← this file
TEMPLATE_VERSION
CHANGELOG.md
settings.json.example             ← optional Stop hook for doc-freshness

docs/                             ← created by commands
├── STRATEGY.md  ARCHITECTURE.md  ROADMAP.md  HANDOFF.md
├── research/                     ← from /sb-research (NEW)
│   ├── INDEX.md                  always loaded by intake-author
│   └── <topic-slug>/
│       ├── INSIGHTS.md           load-bearing for downstream stages
│       ├── BRIEF.md              referenced on demand
│       ├── SOURCES.md
│       └── sources/*.md
├── plans/  reviews/  evals/  decisions/  audits/
└── runs/<run-id>/                ← per-run state (autonomous mode)
    ├── spec.md
    ├── queue.yaml
    ├── manifest.yaml
    ├── escalations.yaml
    └── logs/<task-id>.md

.worktrees/<run-id>/<task-id>/    ← per-task git worktree (autonomous mode)
```

## Pulling updates from upstream

```
curl -L https://github.com/sogwiz/aiflow/archive/refs/heads/main.tar.gz | tar -xz --strip-components=1
```

When the template gets refined:
1. Check your `TEMPLATE_VERSION` against the latest.
2. Read `CHANGELOG.md` entries between those versions.
3. Copy the files you want from a fresh download. Review with `git diff`. Commit.
4. Update your `TEMPLATE_VERSION`.

No automated update tool. Your call what to port.
