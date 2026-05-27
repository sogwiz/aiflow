# Changelog

Versioning is semantic but loose. Bumps signal whether a porting effort is worth your time, not strict API guarantees — this is a starter, not a framework.

- **Major (X.0.0)** — structural change to the workflow (new role, removed command, changed dispatch rules). Read carefully before porting.
- **Minor (1.X.0)** — new agent, new skill, meaningful new capability. Usually worth porting.
- **Patch (1.0.X)** — refinement to existing prompts, wording fixes, small additions. Port if you happen to notice.

## How to use this file

If you cloned at version X, and the template is now at Y, read entries between them. Each entry lists changed files and rationale. You decide what to port.

---

## 2.1.0 — lanes, contract, divergence enforcement, learning metrics

**Why:** four risks identified in v2.0 review — process fatigue, no first-class contract artifact, weak divergence enforcement, no learning-metric path for pre-product work. All addressed without adding new agents or commands.

**Changed files:**

- `.claude/commands/sb-plan.md` — replaced `--quick`-only parsing with **four-lane inference** (Tiny / Small / Normal / High-risk). Lane is inferred from feature description and confirmed before stages run. `--tiny` exits immediately; `--quick` runs stages 1–3 (Small); default runs all 5 (Normal); `--high-risk` runs all 5 plus required `/sb-sanity` before `/sb-work`. Stage 3 now audits `#contract` anchor; Stage 6 surfaces High-risk handoff to `/sb-sanity`.

- `.claude/agents/auditor.md` — requirements mode flags **lane mismatch** if Small/Tiny declared on changes touching sensitive areas. Plan mode adds **Contract integrity check** (BLOCKING): owned files must overlap with affected files, public interface must be specific, acceptance criteria must be observable, boundaries must respect ARCHITECTURE.md.

- `.claude/agents/planner.md` — added **`## Contract` section** to plan template (public interface, ownership boundaries, dependencies, acceptance criteria, merge sequencing). Added **"Designing the contract"** instruction before metrics elicitation — planner must read ARCHITECTURE.md, design boundaries deliberately, check existing plans for conflicts. Added **learning-experiment feature type** to metric proposals (hypotheses tested, time-boxed completion, decision quality, pilot engagement — manual instrumentation accepted).

- `.claude/commands/sb-work.md` — **divergence summary is now mandatory**, even when no divergences occurred. Forces honest reporting; silent matches and silent improvisations look identical otherwise. The summary becomes a forcing function that `/sb-review`'s Pass 0 checks.

- `.claude/agents/reviewer.md` — added **Pass 0: Divergence check** before the four standard passes. Reads plan's `#steps`, `#affected-files`, `#contract` against `git diff`. Undocumented divergences (code that does things the plan didn't list, without a `## Deviations` entry) are blocking findings. Output template adds divergence summary section.

- `.claude/agents/intake-author.md` — added **Pre-product / discovery / R&D** project type to metric menu (customer interviews, prototype usage, qualitative signals, hypothesis tests, time-to-LOI). Telemetry-shaped metrics flagged as wrong for pre-product work.

- `.claude/agents/concerns-auditor.md` — added **learning-metric exception** to the observability rules. Manual instrumentation (interview notes, tester feedback files, decisions log) is acceptable for pre-product work; the traceability check still applies (each metric must trace somewhere) but the destination doesn't have to be telemetry.

- `README.md` — added **artifact chain diagram** alongside the lifecycle diagram (two-screen overview of the system). Added **Lanes: when to use what ceremony** section with the four-lane table. Renamed references from "PRD" framing to "implementation plan" to avoid namespace collision with PM-written PRDs.

- `CLAUDE.md` — workflow section references lanes and divergence summary.

**Migration notes:**

- If you customized `/sb-plan`: this version added a "Step 0: Determine the lane" section before the stages. If your customization touched the stage definitions, reapply on top of the new lane-aware structure.
- If you customized `planner.md`: a new `## Contract` section was added to the plan template, between `## Approach` and `## Success metrics`. Reapply customizations to the new structure.
- If you customized `reviewer.md`: numbering shifted from 1–4 to 0–4 (Pass 0 added). Update any references.
- Existing plans produced by v2.0 don't have `## Contract` sections — they'll generate plan-audit findings on re-audit. Either add Contract sections retroactively or let the findings sit until those features are next touched.

---

## 2.0.0 — lifecycle-stage workflow, cut to 6 commands

**Why:** previous versions accumulated to 9 commands organized loosely. v2.0 reorganizes around product lifecycle (vision → feature → ship → measure) and cuts redundant commands. `/sb-plan` also gets the bug fix for silent stage-skipping that surfaced when running it against a PDF.

**Major changes (breaking):**
- **Cut**: `/sb-onboard`, `/sb-handoff`, `/sb-resume`. Their value is absorbed into `/sb-init` (deep intake works on existing repos) and `/sb-sanity --with-handoff` (drift check + orientation in one).
- **`/sb-init` heavier**: now conducts a four-lens conversation (business, user, product, technical) producing STRATEGY.md, ARCHITECTURE.md, and ROADMAP.md. Add `--quick` for a 6-question fast path producing STRATEGY.md only.
- **`/sb-plan` enforces SURFACE checkpoints**: each stage's output goes to the user before the next stage runs. Fixes the silent-skip bug where the orchestrator would run all five stages in one breath and produce a finished plan without metric elicitation.
- **`/sb-plan` requirements audit blocks on missing metrics**: a feature description without 2-3 named metrics with targets is requirements-incomplete.
- **`/sb-plan` planner has a hard gate on metric elicitation**: planner cannot produce plan output until metric questions have been answered, regardless of how detailed the input requirements are.
- **`/sb-sanity` absorbs handoff and resume**: drift check now covers all the alignment dimensions (STRATEGY→ARCHITECTURE→ROADMAP→Plans→Code→CLAUDE.md). `--with-handoff` flag produces orientation snapshot.
- **Auditor reduced from 5 modes to 3** (requirements, plan, drift): handoff and resume modes folded into drift.
- **`/sb-eval` simplified to one default window (30 days)**: 7-day check available via `--window 7d` flag.

**New artifacts:**
- `docs/ARCHITECTURE.md` — components, data flow, technical risks (deep mode only)
- `docs/ROADMAP.md` — feature horizons with parallelizability annotations (deep mode only)

**Migration from v1.x:** if you had customized agents or commands, the cleanest path is to compare your custom version against the new v2.0 file and re-apply your customizations on top. Specifically:
- If you customized `intake-author`, the structure is now four lenses with explicit question numbering. Rebase customizations onto the new lens structure.
- If you customized `auditor`, modes 1 and 2 are largely unchanged; modes 3-5 collapsed into a single mode 3 (drift) with optional handoff output.
- If you customized `/sb-plan`, the procedure now has explicit SURFACE checkpoints between stages — this is the core bug fix and shouldn't be undone.

If you were actively using `/sb-onboard`, `/sb-handoff`, or `/sb-resume` and want them back, file a note for a future v2.1 — happy to add them back if usage justifies, but the audit said cut.

---

## 1.1.0 — context hygiene commands

Added `/sb-sanity`, `/sb-handoff`, `/sb-resume` as drift detection and recovery commands. Auditor gained drift, handoff, resume modes (3 new modes added).

## 1.0.0 — initial release

Six commands (`/sb-init`, `/sb-onboard`, `/sb-plan` with `--quick`, `/sb-work`, `/sb-review`, `/sb-eval`), six agents, two skills, one optional hook.

---

## Template for future entries

```
## X.Y.Z — short title

**Why:** one-sentence rationale.

**Changed files:**
- `path/to/file.md` — what changed

**Migration notes (if any):** anything porters should watch out for.
```
