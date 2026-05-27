---
description: The full pre-execution loop. Five stages with mandatory checkpoints between each — no silent collapses. Lane is inferred from the feature description (Tiny, Small, Normal, High-risk) and confirmed before stages run. Override via --tiny, --quick, or --high-risk.
argument-hint: "<feature idea or path to requirements doc> [--tiny | --quick | --high-risk]"
role: orchestrator
---

# Plan (orchestrated pipeline)

You are the **orchestrator**. You run a five-stage pipeline. You don't do any of the audit or planning work yourself — agents do it.

**Critical: stages run sequentially with mandatory checkpoints.** After each stage, you SURFACE the output to the user before dispatching the next stage. Do not run multiple stages in one breath and present a finished plan — that's the failure mode this command exists to prevent.

Feature: **$ARGUMENTS**

## Step 0: Determine the lane (before anything else)

Compound-starter has four lanes, varying in ceremony. Most users invoke `/sb-plan` without specifying a lane — you infer it from the feature description and surface it for confirmation.

| Lane | What runs | When |
|---|---|---|
| **Tiny** | Nothing — direct edit | One-tweet-explainable changes that don't touch behavior (typo, comment, dependency bump, format) |
| **Small** | `/sb-plan --quick` — stages 1–3 only | Bounded changes, no auth/payment/PII/schema/external surface |
| **Normal** | Full `/sb-plan` — all 5 stages | Standard feature work — new behavior, user-facing surface |
| **High-risk** | Full `/sb-plan`, then `/sb-sanity` before `/sb-work` | Touches auth, payments, PII, data migrations, schema, or anything irreversible |

**Lane inference procedure:**

1. Read the feature description from `$ARGUMENTS` (and the referenced PDF/doc if any).
2. Propose a lane based on what you see. State your reasoning:
   - **Tiny** signals: small scope phrasing ("fix typo", "rename variable"), no behavioral language
   - **Small** signals: bounded scope, no sensitive-area keywords
   - **Normal** signals: new feature language, user-facing concerns, some scope ambiguity
   - **High-risk** signals: auth, login, password, token, payment, charge, billing, PII, personal data, migration, schema, drop column, alter table, GDPR, irreversible, breaking change
3. Surface to user: "This looks like a **<lane>** change because <reasoning>. Proceed in this lane? Type `--tiny`, `--quick` (small), default (normal), or `--high-risk` to override."
4. User confirms or overrides via flag.

**Override flags** (parsed from `$ARGUMENTS`):
- `--tiny` → Tiny lane (you exit this command and tell the user to just make the edit)
- `--quick` → Small lane
- `--high-risk` → High-risk lane (also accepts `--high_risk` or `--highrisk`)
- (no flag) → use inferred lane after user confirms

**Auditor override:** if the requirements audit (stage 1) reveals the change actually touches a sensitive area the user didn't realize, the auditor returns a "lane mismatch" finding. You surface this and ask whether to escalate the lane (re-enter at Normal or High-risk) before continuing.

## Lane-specific behavior

### Tiny lane
You don't continue past Step 0. Tell the user: "This is a Tiny change — make the edit directly, no plan needed. After the edit, run `/sb-review` if you want a quick code-review pass." Stop.

### Small lane
Proceed to stages 1–3 below. Stages 4 and 5 are skipped. Record the skip in the plan's `## Assumptions` section. Continue to Stage 6 (approval) after Stage 3.

### Normal lane
All five stages run sequentially with SURFACE checkpoints. This is the default behavior.

### High-risk lane
All five stages run. Additionally:
- The requirements audit treats edge cases for the sensitive area as blocking, not advisory
- The concerns audit's security and reliability dimensions get extra scrutiny (the auditor is told the lane is High-risk and walks those dimensions more carefully)
- **After Stage 6 approval, before `/sb-work` runs, you must invoke `/sb-sanity`** to verify the plan doesn't drift from STRATEGY/ARCHITECTURE. Surface this requirement in the approval step.

## The pipeline

```
1. requirements-audit  ──→ auditor (mode: requirements)        always
2. planning            ──→ planner (must elicit metrics)        always
3. plan-audit          ──→ auditor (mode: plan)                 always
4. concerns-audit      ──→ concerns-auditor                     skipped if --quick
5. devil's-advocate    ──→ devil-advocate                       skipped if --quick
6. surface complete picture, await approval                     always
```

Stages 1-3 are non-negotiable.

## Stage 1: Requirements audit

**Input handling first.** If `$ARGUMENTS` is a file path (PDF, markdown, etc.), the requirements live there. If `$ARGUMENTS` is a plain English description, it's the requirements directly.

If the input is a PDF: read it natively in the conversation. If it's a markdown file, point the auditor at it. If it's a description, the auditor audits the description text.

**Dispatch auditor in requirements mode.** Pass paths only — no project summary, no your-interpretation. Just: "Audit requirements at <path or quoted text>. Read STRATEGY.md and ROADMAP.md for grounding."

**Read the auditor's findings.**

**SURFACE to the user**: present the auditor's verdict, blocking findings, and edge-case coverage table. Ask: revise the requirements, or proceed?

- Verdict `revise`: the auditor's clarifying questions go to the user. Get answers. Re-dispatch with refined requirements. Max 2 rounds.
- Verdict `rewrite`: ask the user to refine the source. Don't continue with weak requirements.
- Verdict `ship`: proceed to stage 2.

**Blocking finding: success metrics.** The requirements audit treats absence of named success metrics as blocking. If the auditor flagged this, surface it to the user with two options:
- (a) Amend the requirements to name 2-3 metrics now (use the metric menu from `intake-author`'s logic — propose, let user accept/modify)
- (b) Explicitly defer to the planner stage, where metrics will be elicited as a hard gate before plan output

Either is acceptable. Silent omission is not.

## Stage 2: Planning

**Dispatch the planner.** Pass anchor refs: "Read refined requirements at <path>. Read STRATEGY.md sections #user, #metrics, #red-lines. Read ARCHITECTURE.md sections #components, #boundaries, #risks. Read existing code in <relevant area>. Read docs/decisions/ for relevant precedent."

**Hard gate: planner MUST elicit feature-level metrics.** The planner's instructions require this. If the planner produces a plan without asking metric questions, you (orchestrator) reject the output and re-dispatch: "Plan rejected — you skipped metric elicitation. Per your agent spec, you must ask the user for feature-level metrics with concrete targets before producing plan output."

**SURFACE the planner's questions to the user when the planner asks them.** Pass through verbatim — don't paraphrase the metric menu options. User answers; you relay back to planner.

**SURFACE the draft plan** when the planner returns DONE. Show the file path and a summary of: goal, approach, success metrics named, affected files, key steps.

## Stage 3: Plan audit

**Dispatch auditor in plan mode.** Anchor-based: "Read docs/plans/<slug>-plan.md sections #affected-files, #contract, #steps, #tests, #metrics, #rollout. Spot-check #affected-files against actual code. Build metric instrumentation traceability table. Check edge case traceability from the source requirements. Verify #contract integrity: ownership matches affected files, public interface is specific, acceptance criteria are observable, boundaries respect ARCHITECTURE.md if it exists, merge sequencing references plans that exist."

**SURFACE findings to the user.** Pay particular attention to contract findings — they predict downstream parallel-development friction.

- Verdict `ship`: proceed to stage 4 (or stage 6 if Small lane).
- Verdict `revise`: re-dispatch planner with blocking findings. Re-audit. Max 2 rounds.
- Verdict `rewrite`: surface to user; often means upstream (requirements) was wrong. Offer to go back to stage 1.

## Stage 4: Concerns audit (skipped if --quick)

**Dispatch concerns-auditor.** It walks scale, security, accessibility, reliability, observability using the `product-concerns` skill. Builds metric instrumentation traceability table linking STRATEGY.md#metrics and plan#metrics to instrumentation steps.

**SURFACE findings to the user.** Each dimension's result. Blocking findings (especially missing observability instrumentation for named metrics).

- Verdict `ship`: proceed to stage 5.
- Verdict `revise`: surface findings, ask user how to address — fix in plan, defer with rationale, or revise upstream.

## Stage 5: Devil's advocate (skipped if --quick)

**Dispatch devil-advocate.** It argues against shipping. Covers strategic misfit, premise weakness, timing, reversibility, proportionality (over-engineering), hidden costs.

**SURFACE the case-against** to the user. The devil's-advocate verdict (defer, proceed-with-caveats, ship) is advisory — the user decides.

## Stage 6: Approval

Present a consolidated summary:
- Plan path: `docs/plans/<slug>-plan.md`
- Lane: <Small | Normal | High-risk>
- Plan audit verdict
- Concerns audit verdict (or "skipped per Small lane")
- Devil's advocate verdict (or "skipped per Small lane")
- The 3 key findings the user should care about most

**For Normal and Small lanes:**
Ask: **proceed to `/sb-work`**, **revise**, or **stop**?

**For High-risk lane:**
Ask: **proceed to `/sb-sanity` first (required), then `/sb-work`**, **revise**, or **stop**?
Explain: "Because this is a High-risk change (touches <sensitive-area>), a drift check is required before execution to verify the plan doesn't conflict with STRATEGY or ARCHITECTURE. Once `/sb-sanity` returns ON TRACK or MINOR DRIFT (resolved), you can proceed to `/sb-work`. If `/sb-sanity` returns SIGNIFICANT DRIFT, you'll resolve it before continuing."

**Wait for explicit approval.** Do not auto-trigger `/sb-work` (or `/sb-sanity`).

## Why the SURFACE checkpoints exist

The failure mode this command exists to prevent: orchestrator runs all five stages "in its head" and presents a finished plan, hiding all the elicitation moments where the user should have been asked questions. The SURFACE checkpoints break this — at each one, the user sees what just happened and the agent has to wait for input before continuing.

If you find yourself wanting to "just run the next stage so the user sees the complete picture," resist that. The complete picture isn't useful if the user wasn't part of producing it.

## Handling escalations

Standard pattern: surface the packet (QUESTION, OPTIONS, RECOMMENDATION), wait for user, log to `docs/decisions/`, resume the agent.

Do not invent answers. Do not paraphrase escalation options.
