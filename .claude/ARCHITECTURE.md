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

## Escalation contract (three classes)

Agents classify uncertainty:

- **Judgment** — multiple defensible options, depends on your values → **escalate**
- **Risk** — irreversible or high blast radius → **escalate**
- **Routine** — any reasonable choice works → **pick and document under `## Assumptions`**

Every escalation includes a recommendation with rationale so you can disagree with the *reasoning*, not just pick a letter. Decisions log to `docs/decisions/` and future agents read them as precedent.

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
