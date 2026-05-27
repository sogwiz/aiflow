# Compound-starter

A lightweight Claude Code workflow template: vision → feature → ship → measure → repeat.

**Six commands, six agents, two skills, one optional hook.** Read this file in 5 minutes and you'll know when to run what.

## The workflow at a glance

Six commands, organized by lifecycle stage:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    Vision                Build the feature           After ship │
│    ───────              ─────────────────           ──────────  │
│                                                                 │
│    /sb-init             /sb-plan <feature>          /sb-eval    │
│         │                    │                          ↑       │
│         │                    │                          │       │
│         ↓                    ↓                          │       │
│    STRATEGY.md          docs/plans/...                  │       │
│    ARCHITECTURE.md           │                          │       │
│    ROADMAP.md                ↓                          │       │
│                          /sb-work                       │       │
│                              │                          │       │
│                              ↓                          │       │
│                          /sb-review                     │       │
│                              │                          │       │
│                              └──────── ship ────────────┘       │
│                                                                 │
│    Anytime:  /sb-sanity (drift check)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## The artifact chain

What produces what, end to end:

```
Assignment / PDF / idea
     │
     ↓
/sb-init   →   STRATEGY.md  +  ARCHITECTURE.md  +  ROADMAP.md
     │
     ↓
/sb-plan   →   docs/plans/<feature>-plan.md   (with #contract section)
     │
     ↓
/sb-work   →   shipped code
     │
     ↓
/sb-review →   docs/reviews/<feature>-review.md
     │
     ↓
/sb-eval   →   docs/evals/<feature>-<window>.md
```

The plan is the **implementation plan** — not a PRD. Requirements live upstream (in STRATEGY.md and the feature description); the plan is how we'll build what the requirements call for. Each plan has a `#contract` section so other features can depend on it without reading the body — this is what makes parallel development safe.

## What each command does

**Vision (run once at project start, refresh occasionally):**
- `/sb-init` — heavy four-lens intake (business, user, product, technical). 30-60 minutes of real conversation. Produces STRATEGY.md, ARCHITECTURE.md, and a ROADMAP.md with features annotated for parallelizability. Add `--quick` for a 6-question fast path producing only STRATEGY.md (use when starting small).

**Per-feature loop:**
- `/sb-plan <feature>` — five-stage pipeline: requirements audit → planner (must elicit metrics + design contract) → plan audit → concerns audit (scale, security, accessibility, reliability, observability) → devil's-advocate (argues against shipping). Each stage SURFACES findings to you before the next runs. Lane is inferred from the feature description; override via `--tiny`, `--quick`, or `--high-risk`.
- `/sb-work` — execute the approved plan. Stops if reality diverges. Produces a mandatory divergence summary at the end.
- `/sb-review` — code review with five passes (divergence check + correctness, edge cases, security, simplicity) in sequence.

**After ship:**
- `/sb-eval <feature>` — measure actual outcomes against the named metrics in STRATEGY and the plan. Default window 30 days; `--window 7d` for shipping-quality check.

**Anytime:**
- `/sb-sanity` — independent drift check by a fresh-context agent. Reads STRATEGY, ARCHITECTURE, ROADMAP, recent plans, code, and CLAUDE.md and reports alignment. Add `--with-handoff` to also produce a `docs/HANDOFF.md` orientation snapshot.

## Lanes: when to use what ceremony

Not every change needs the full pipeline. Compound-starter has four lanes; `/sb-plan` infers your lane from the feature description and asks you to confirm before running.

| Lane | What runs | When to use |
|---|---|---|
| **Tiny** | No `/sb-plan`. Direct edit. | One-tweet-explainable changes that don't touch behavior: typos, comments, dependency bumps, formatting. |
| **Small** | `/sb-plan --quick` — stages 1–3 only (skips concerns audit + devil's advocate). | Bounded changes with no auth/payment/PII/schema/external surface. Small features and bug fixes. |
| **Normal** | Full `/sb-plan` — all 5 stages. | Standard feature work — new behavior, user-facing surface, anything you'd describe as a "feature" to a non-engineer. |
| **High-risk** | Full `/sb-plan`, then `/sb-sanity` before `/sb-work`. | Touches auth, payments, PII, data migrations, schema changes, or anything irreversible. |

The lane is your declaration, but the requirements-auditor can reject it. If you say "Small" on a change that touches auth, you'll get a lane-mismatch finding and a prompt to escalate. The lanes are a fast path, not a way to silently downgrade scrutiny.

## Install

### New project
1. On GitHub, click **"Use this template"** → "Create a new repository."
2. Clone locally. Open in Claude Code.
3. Run `/sb-init` (or `/sb-init --quick` if you want light intake).

### Drop into existing project
1. Copy `.claude/`, `CLAUDE.md`, `settings.json.example`, `TEMPLATE_VERSION`, `CHANGELOG.md` into the repo.
2. If the repo already has a `CLAUDE.md`, rename ours to `CLAUDE.md.template` and merge selectively.
3. Run `/sb-init` — even on existing repos, the deep intake gives you STRATEGY/ARCHITECTURE/ROADMAP grounded in what already exists.

## The strong opinions

These are the design calls worth knowing about. Full rationale in `.claude/ARCHITECTURE.md`.

1. **Roles are strict.** Orchestrators dispatch; implementors produce; auditors critique with read-only tools; humans decide. Orchestrators never edit artifacts or do agent work themselves.
2. **One-hop dispatch.** Agents don't dispatch agents. Orchestrator is the single source of truth for what happened.
3. **Audit independence.** Auditors see the artifact, not the author's reasoning. Fresh context. No pre-digestion.
4. **SURFACE checkpoints in `/sb-plan`.** Each stage's output goes to you before the next runs. Prevents silent collapse of the pipeline into a fait accompli.
5. **Metrics must be instrumented.** Requirements audit blocks if no metrics are named. Plan audit blocks if named metrics have no instrumentation step. Eval reads the instrumented values.
6. **Menu-driven elicitation.** Vision questions, metrics questions, and similar use propose-then-confirm with concrete options — never freehand "what are your metrics" cold.
7. **Compose with built-ins, never replace.** `/sb-init` invokes Claude Code's built-in `/init` first to scaffold CLAUDE.md, then layers the deep intake.
8. **Roadmap-driven parallelism.** ROADMAP.md annotates features with which architecture components they touch. Features touching different components can be built concurrently.
9. **Decision routing keeps momentum without hiding risk.** Agents decide tactical details silently, decide and log technical choices in the plan, and escalate architectural/strategic/risky choices to you.
10. **Decisions log is the system's memory of your judgment.** Human-resolved architectural, strategic, and risk decisions go in `docs/decisions/`. Repeated decisions promote to CLAUDE.md.
11. **Eval is the truth source.** Strategy is aspirational until measured. The 30-day eval catches premise problems that the build loop misses.

## What we cut to get here

Earlier versions had `/sb-onboard`, `/sb-handoff`, `/sb-resume` — three additional commands for joining existing projects, alignment testing, and orientation snapshots. They worked but were thin slices of what `/sb-sanity` already does. Cut in v2.0:
- `/sb-onboard` → just run `/sb-init` (the deep intake works on existing repos)
- `/sb-handoff` → absorbed into `/sb-sanity` (the drift check is the alignment test)
- `/sb-resume` → use `/sb-sanity --with-handoff` (drift check + orientation snapshot in one)

If you find yourself missing one of these, that's data for a future version. Until then, fewer commands wins.

## File layout

```
.claude/
├── ARCHITECTURE.md              ← workflow contract (read this second, 5 min)
├── commands/                    ← orchestrators
│   ├── sb-init.md
│   ├── sb-plan.md
│   ├── sb-work.md
│   ├── sb-review.md
│   ├── sb-eval.md
│   └── sb-sanity.md
├── agents/                      ← implementors + auditors
│   ├── intake-author.md
│   ├── planner.md
│   ├── auditor.md               (3 modes: requirements / plan / drift)
│   ├── concerns-auditor.md
│   ├── devil-advocate.md
│   └── reviewer.md
└── skills/
    ├── product-concerns/SKILL.md   (checklists for the concerns-auditor)
    └── doc-freshness/SKILL.md      (proposes doc updates at eval time)

CLAUDE.md                        ← project context (fill in TODOs)
README.md                        ← this file
TEMPLATE_VERSION
CHANGELOG.md
settings.json.example            ← optional Stop hook for doc-freshness

docs/                            ← created by commands
├── STRATEGY.md  ARCHITECTURE.md  ROADMAP.md  HANDOFF.md
├── plans/  reviews/  evals/  decisions/  audits/
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
