# Architecture (workflow contract)

Six commands, six agents, two skills, one optional hook. Read in 5 minutes.

## Lifecycle stages

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
│    Anytime:  /sb-sanity (drift check + optional handoff doc)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Roles

| Role | Job | Hard rules |
|---|---|---|
| **Orchestrator** | Dispatch agents. Never edit. Never audit. SURFACE outputs to user between stages. | Lives in `.claude/commands/`. |
| **Implementor** | Produce an artifact. Full tools. | Returns DONE or ESCALATE. Doesn't dispatch others. |
| **Auditor** | Critique an artifact. | Read-only tools. Fresh context. Returns findings, never edits. |
| **Devil's advocate** | Argue against shipping. | One run, between plan and code. Read-only. |
| **Reviewer** | Critique code after implementation. | Four perspectives in one agent, sequential. |
| **Human** | Resolve escalations, set direction. | You. |

## Dispatch flow

```
User invokes /command
     ↓
Orchestrator (command)
     ↓
Agent (implementor or auditor)
     ↓
     ├── DONE → orchestrator → SURFACE to user → next step
     └── ESCALATE → orchestrator → Human → decision logged → resume agent
```

One-hop only. Agents don't dispatch agents. Orchestrators never do agent work themselves.

## Decision routing

Agents classify decisions by blast radius:

- **Tactical** — naming, imports, formatting, local refactors → decide silently
- **Technical** — library options, implementation pattern, cache shape, test strategy → decide and log under the plan's `## Decision log` (or the relevant generated artifact outside `/sb-plan`)
- **Architectural** — component boundary, public interface, dependency direction, data model, new external service → escalate
- **Strategic / risk** — scope, product direction, user promise, irreversible or high-blast-radius action → escalate

Use this rubric in order:

1. **Could this change the product promise, user scope, roadmap priority, red line, or success metric?** Strategic / risk.
2. **Could this be hard to reverse, expose data, affect auth/payments/PII/schema/migrations, or change behavior outside the current feature?** Strategic / risk.
3. **Could another feature reasonably depend on this boundary, interface, data model, dependency direction, or external service choice?** Architectural.
4. **Is this a meaningful implementation choice inside approved boundaries, with reversible consequences?** Technical.
5. **Is this only local naming, organization, formatting, or glue code with no behavioral or interface impact?** Tactical.

Every escalation includes a recommendation with rationale so you can disagree with the *reasoning*, not just pick a letter. Human-resolved architectural, strategic, and risk decisions log to `docs/decisions/` and future agents read them as precedent. Technical decisions stay close to the work unless repeated enough to promote into `CLAUDE.md`.

## The vision-to-feature pipeline

```
/sb-init <description or PDF>          [DEEP MODE: 30-60 min conversation]
  └── four lenses: business, user, product, technical
       └── produces STRATEGY.md, ARCHITECTURE.md, ROADMAP.md

/sb-init --quick                       [QUICK MODE: 6 questions, ~5 min]
  └── produces STRATEGY.md only; flags what was skipped

──────────────────────────────────────────────────────────────────

/sb-plan <feature>                     [pre-execution pipeline]
  ├── Stage 1: requirements audit       SURFACE
  ├── Stage 2: planner (must elicit metrics)  SURFACE
  ├── Stage 3: plan audit               SURFACE
  ├── Stage 4: concerns audit           SURFACE  (skipped if --quick)
  ├── Stage 5: devil's-advocate         SURFACE  (skipped if --quick)
  └── Stage 6: user approval

/sb-work                               [execute the approved plan]
/sb-review                             [4-pass code review]
/sb-eval <feature>                     [post-ship measurement]
```

**SURFACE checkpoints are mandatory.** Each stage's output goes to the user before the next stage runs. This prevents the failure mode where all five stages run silently and a finished plan appears without elicitation.

## Namespace: `sb-` prefix

All custom commands use `sb-` ("super-builder") to avoid collisions with Claude Code's built-ins (`/init`, `/plan`, `/clear`, etc.). When a built-in does part of what we need, our command **composes** with it rather than replacing it. The clearest case is `/sb-init`, which invokes built-in `/init` first to scaffold `CLAUDE.md`, then layers the deep intake on top.

## Roadmap-driven parallelism

The ROADMAP.md produced by `/sb-init` annotates features with which architecture components they touch. Features touching different components can be built in parallel; features touching the same component must be sequenced. This is the practical mechanism for the "parallelize feature development" goal — it falls out of having an architecture that defines clean component boundaries.

## Why this is small on purpose

A new engineer joining your team should read this file in 5 minutes and understand the workflow. If something doesn't fit that budget, it gets merged or cut. The previous version had nine commands; we cut to six. The pattern: when the value of a piece can be delivered by extending an existing piece rather than adding a new one, extend.
