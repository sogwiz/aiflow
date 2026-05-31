---
description: Domain research stage upstream of /sb-init. Default is a lean, cost-disciplined path (single question, snippet-first, source cap 8, lazy per-source notes); --thorough opts into the heavier deep-research skill with adversarial cross-checking.
argument-hint: "<topic-slug> [inputs] [--thorough] [--max-sources N] [--no-web] [--use-only-provided] [--view]"
role: orchestrator
---

# Research (domain grounding)

You are the **orchestrator**. Domain research often belongs *before* product inception — building a learning app benefits from grounding in learning science before strategy questions are even framed. `/sb-research` runs that research and structures the output so `/sb-init` (and later `/sb-spec`, `/sb-plan`) can consume it without loading the entire corpus into conversation context.

**Two modes:**

- **Cheap mode (default)** — single research pass, snippet-first web reads (full pages only for top 3 sources), hard source cap of 8, no adversarial cross-checking, synthesis with sonnet. Sized to produce the load-bearing artifact (`INSIGHTS.md`) at the lowest cost that still meets downstream needs.
- **Thorough mode (`--thorough`)** — invokes the built-in `deep-research` skill with full fan-out, source cross-checking, and adversarial verification. Use only when the stakes warrant it: regulatory work, high-investment strategic bets, or research the user explicitly wants peer-reviewed.

The default is cheap because downstream stages (`intake-author`, `planner`) read `INSIGHTS.md` (~400 words) — not the full corpus. Paying for adversarially-verified, multi-fan-out research and then summarizing to 400 words is paying for verification that gets thrown away. Cheap-by-default keeps `/sb-research` as a routine grounding step, not a flagship expense.

Args: **$ARGUMENTS** — first token is `<topic-slug>` (lowercase kebab-case, 3-40 chars). Rest is the research question + any input pointers + optional flags.

## Why this exists

Without structured output, research artifacts get dumped into the next stage's context unfiltered — easily 50-300K tokens of paper summaries and repo evaluations. The intake-author then either ignores them (waste) or chokes on them (degradation). With structured output, downstream stages read 400 words of INSIGHTS, reference the BRIEF on demand, and only crack open source notes when something specific is challenged. Same intellectual leverage, 10× the context efficiency.

## Cost discipline (the load-bearing change)

`/sb-research` is a *routine* grounding step, not a flagship expense. Earlier versions defaulted to `deep-research` for everything — fan-out web searches, full-page fetches across 10-20 sources, adversarial verification per claim, multi-pass synthesis. For a question whose downstream consumer reads 400 words, that's burning tokens on rigor that gets thrown away.

**Defaults in cheap mode (always-on unless `--thorough`):**

| Lever | Cheap default | Why |
|---|---|---|
| Web fan-out | Skipped if the user provided ≥5 corpus inputs; otherwise capped at 3-5 targeted searches | Search snippets cost ~200-400 tokens each; full pages 5-30k. Snippets first, full only for top picks. |
| Source cap | **Hard cap: 8 total** (user inputs count toward the cap) | Quality plateaus past ~5 strong sources. More sources mean more re-reads at synthesis time. |
| Full-page fetches | Only the top 3 by relevance score from snippet pass | The other 5 contribute via snippets — enough to cite, not enough to bloat. |
| Adversarial cross-checking | **Off by default**; on with `--thorough` | Verification is expensive (each claim → multiple sources → re-read). Cheap mode marks contested items via the `confidence` map rather than resolving them. |
| Synthesis model | sonnet | Single-pass synthesis of pre-summarized snippets — opus's extra capacity is wasted here. |
| Per-source notes | **Lazy** — only written for sources cited in INSIGHTS | Eagerly writing notes for every source touched was a major hidden cost. Uncited sources stay in SOURCES.md as references only. |
| INSIGHTS hard cap | **400 words** | Forces synthesis to be load-bearing. Was 300-700 — the larger limit invited padding. |
| BRIEF hard cap | **1 page (~600 words)** | Down from 1-2 pages. The brief is rarely loaded; cap it. |

**Estimated savings:** ~50-70% token reduction vs. the old `deep-research`-driven default for a typical research question.

**Flags (override the defaults):**

- `--thorough` — invoke the `deep-research` skill with full fan-out + adversarial verification. Use when stakes warrant peer-reviewed rigor.
- `--max-sources N` — raise (or lower) the hard cap from 8. Capped at 15 to prevent runaway.
- `--no-web` — never call web search. Useful when the user has provided the entire corpus or you're operating offline.
- `--use-only-provided` — synonym for `--no-web`, makes intent louder when the user is deliberately scoping to their own sources.
- `--view` — after research completes, compose with `/sb-view <slug>` to render an HTML view at `.local/research-views/<slug>/` for human consumption. The MD files (under `docs/research/`) are for agent consumption; the HTML view is for you.

## Pre-flight

1. Parse `<topic-slug>`. Validate format: lowercase kebab-case. If invalid or missing, ask the user for a slug (e.g., `learning-science`, `competitor-landscape`, `motivation-by-age`).
2. Parse flags from `$ARGUMENTS` — `--thorough`, `--max-sources N`, `--no-web`, `--use-only-provided`.
3. Check `docs/research/<topic-slug>/` exists. If yes: "Topic exists. Augment, overwrite, or pick a new slug?" Augmenting means a new round adds sources and updates INSIGHTS/BRIEF without losing prior source notes.
4. Identify input sources from `$ARGUMENTS`:
   - PDF paths (anything ending `.pdf`) — read natively
   - URLs (http/https) — fetched with snippet-first pass (cheap mode) or full pages (thorough mode)
   - Local file paths (e.g., requirement briefs) — read directly
   - Repo paths (local) or repo URLs — evaluated as competitor/prior-art artifacts
   - Freeform text — the research question itself
5. **Input volume guard.** If the user provided more than 5 user inputs in one topic, **refuse and ask to split**: "12 inputs is too many for one topic. Splitting into 2-3 topic-slugs produces sharper INSIGHTS and stays under context budget. Which sources go where?" This is not a soft suggestion — it's the cheapest correctness lever in the whole command.

## Procedure

### Step 1: Frame the research question

If the user didn't provide a tight research question, ask one short question to scope it. Examples:
- "What's the decisive question you want answered?" (single best question)
- "Which decisions does this research need to inform — strategy, architecture, or both?"

Don't run open-ended research without a question. Open-ended produces sprawl; the structured-output requirement won't save you from a bad scope. **One question per `/sb-research` invocation.** Multiple questions = multiple topic-slugs.

### Step 2 (cheap mode, default): Lean research path

This is the default. **No `deep-research` fan-out**; you do the research inline with these stages, each bounded by the cost-discipline caps above.

**2a. Read user-provided sources first.** PDFs, local files, repos. For each, write a one-paragraph internal summary (kept in working context only — NOT written to a file yet). If this fills the source cap (≥5 from user), skip 2b entirely and go to 2c.

**2b. Targeted web pass (skipped if `--no-web` or `--use-only-provided`).**
- Generate 3-5 targeted search queries from the question. Not "learning science" — "spaced repetition retention K-12, evidence", "intrinsic motivation age 11+ developmental".
- Fetch snippets only for all results. Score by relevance to the question.
- Fetch full pages for **top 3** unique sources.
- Discard the rest from working memory — only the top picks plus the snippets-with-citations.

**2c. Synthesis pass (sonnet).**
- Read user-input summaries + top-3 full pages + snippets.
- Produce: `INSIGHTS.md` (≤400 words), `BRIEF.md` (≤1 page), `SOURCES.md` (all sources touched, with snippet-only marked as such).
- For sources cited in INSIGHTS, write per-source notes lazily — only those. Uncited sources stay listed in SOURCES.md but no per-source file.

**2d. Confidence map.** Mark items as `high` / `contested` / `speculative` based on agreement across sources. No adversarial verification — just note when sources disagree. Cheap mode is honest about uncertainty rather than expensive about resolving it.

### Step 2 (thorough mode, `--thorough`): Dispatch deep-research

When `--thorough` is set, invoke the built-in `deep-research` skill with:

```
Question: <the framed research question>
Inputs to read: <PDF paths, URLs, repo locations the user provided>
Source cap: <max_sources, default 8 (raised by --max-sources)>
Output requirement: structured per /sb-research convention — return a JSON-like packet with:
  - insights:    array of structured insight objects (see schema below)
  - brief:       1-page exec summary, markdown
  - sources:     array of {title, type, url_or_path, key_claims, our_notes}
  - confidence:  {high: [...], contested: [...], speculative: [...]}
Context budget: prioritize the user's named inputs (PDFs, repos). Web search is for filling gaps, not replacing the named corpus.
Adversarial verification: enabled — cross-check each load-bearing claim against ≥2 sources.
```

The deep-research skill does the actual work — fan-out searches, source-by-source reading, adversarial verification. This command's value is the output discipline AND the cost-discipline caps it passes down.

**When thorough is actually right:** regulatory/compliance research, strategic bets above a stated investment threshold, claims you'll cite to outside parties. For the typical "ground the strategy in domain knowledge" use case, cheap mode is the right call. If you're not sure, default to cheap and re-run with `--thorough` if INSIGHTS comes back thin.

### Step 3: Write the structured output

When deep-research returns, write four files. **All four are mandatory.** No file = no research; the directory is incomplete and downstream stages will skip it.

#### `docs/research/<topic-slug>/INSIGHTS.md` — the load-bearing artifact

```markdown
# Insights: <topic>

_Researched: <date>_  _Question: <verbatim>_

## Decisions this research implies
- **<short heading>** — <1-2 sentences. What this means for the product. Cite sources by ref number, e.g., [S3, S7].>
- ...

## Constraints discovered
- **<short heading>** — <1-2 sentences. A red line, must-have, or non-negotiable surfaced by the research. Cite.>
- ...

## Open questions for the user
- <Things the research cannot answer. Decisions the human must make. Be specific.>
- ...

## Confidence map
- **High confidence:** <what the literature broadly agrees on>
- **Contested:** <where sources disagree, with the disagreement summarized in one line>
- **Speculative:** <interesting but thin>

## Sources index
- [S1] <title> — `sources/<filename>.md`
- [S2] ...
```

The "Decisions this research implies" + "Open questions for the user" sections are what intake-author and the planner read at sb-init and sb-spec time. Without them, this is just storage.

**Rules for the INSIGHTS file:**
- **400 words total, hard cap.** If you can't fit it, the research wasn't synthesized — fix that, don't expand the file. Cheap mode optimizes for sharp INSIGHTS; padded files defeat the purpose.
- Every insight cites a source. Uncited = assertion, not insight; demote or drop.
- "Open questions for the user" must be answerable in a single intake-author turn. "Define your product" is not an open question; "Choose between intrinsic and extrinsic motivation as the primary engagement loop" is.

#### `docs/research/<topic-slug>/BRIEF.md` — the executive summary

```markdown
# Brief: <topic>

_Researched: <date>_  _Question: <verbatim>_

## Summary
<3-5 paragraphs. The synthesis. Reads as a coherent narrative, not a bullet list of facts.>

## Key findings (with citations)
<5-15 findings, each 2-4 sentences, cited.>

## Methodology
<1-2 paragraphs. What was read, what was searched, what was excluded and why.>

## Limitations
<What this research can't answer; where the sources are weakest.>
```

**1 page (~600 words), hard cap.** Downstream stages don't load this by default — they reference it when INSIGHTS surfaces a question that needs more. If your synthesis runs longer than 1 page, the question was too broad for one `/sb-research` run — split into topic-slugs.

#### `docs/research/<topic-slug>/SOURCES.md` — the source index

```markdown
# Sources: <topic>

| Ref | Title | Type | Location | Date | Read fully? |
|---|---|---|---|---|---|
| S1 | <title> | paper / repo / article / docs / interview | `sources/s1-<slug>.md` | <year> | ✅/⚠️ |
| S2 | ... | ... | <url or path> | ... | ... |
```

#### `docs/research/<topic-slug>/sources/<sN-slug>.md` — per-source notes (LAZY)

**Only write a per-source note for sources cited in INSIGHTS.md.** Uncited sources stay listed in SOURCES.md (with their title, type, and URL/path) but get no per-source file. This is the largest hidden cost in the old eager-write design.

When you do write one, the format is:

```markdown
# [S<N>] <source title>

**Type:** paper / repo / article / docs / interview
**Original location:** <url or path>
**Read on:** <date>
**Read depth:** full | snippets-only       ← snippet-only sources never get a per-source file; this row only appears when full was read

## What it claims
<bulleted, specific>

## Quoted passages we relied on
> "<quote>" — page/section ref

## Our evaluation
<one paragraph: is this a strong source for this question? Where do we trust it, where do we hedge?>

## Cross-references
- Agrees with: [S3]
- Disagrees with: [S5] on <point>       ← in cheap mode, cross-references are populated only when observed during synthesis, not actively probed
```

### Step 4: Update or create `docs/research/INDEX.md`

This is the directory of all research topics — always loaded by intake-author, so it must stay tiny.

```markdown
# Research index

| Topic | Question | Researched | Insights | Brief |
|---|---|---|---|---|
| `learning-science` | "What does learning science say about retention in self-paced apps?" | 2026-05-29 | `learning-science/INSIGHTS.md` | `learning-science/BRIEF.md` |
| `motivation-by-age` | "How do intrinsic vs extrinsic motivations weight across ages 6-18?" | 2026-05-29 | ... | ... |
```

Append a row; don't rewrite the file. If the topic already exists (augmentation), update its row.

### Step 5: SURFACE the output

Print:

```
Research complete: <topic-slug>
Mode:        <cheap | thorough>
Sources:     <n total = n full + n snippets-only>   (cap: <max_sources>)
Web pass:    <skipped | n searches, top-3 full-fetched>

INSIGHTS.md  (<word count>/400):  docs/research/<topic-slug>/INSIGHTS.md
BRIEF.md     (<word count>/600):  docs/research/<topic-slug>/BRIEF.md
SOURCES.md   (<n> sources):       docs/research/<topic-slug>/SOURCES.md
sources/     (<n> notes — cited only):  docs/research/<topic-slug>/sources/
INDEX.md updated.

Top 3 decisions implied:
1. <heading from INSIGHTS>
2. ...
3. ...

Open questions you'll need to answer in /sb-init:
- <list>

Cost notes (cheap mode only):
- If INSIGHTS feels thin or contested-items dominate, re-run with --thorough for the same topic-slug (augments existing artifacts; doesn't lose snippet-only references).
- If a specific source's claim is load-bearing for a downstream decision, the per-source note can be promoted from snippet to full read on demand.

Next: /sb-research <next topic> if more research is needed, or /sb-init to start product inception with these insights grounded.

### Step 6 (if `--view` was passed): Render HTML view

After writing the MD artifacts, compose with `/sb-view <topic-slug>`. The view renders this topic's HTML pages AND refreshes the cross-topic dashboard at `.local/research-views/index.html`. The view is gitignored — it's for human consumption, not the agents.

If the renderer escalates (missing source, malformed INSIGHTS), surface but do not retry — the MD artifacts are valid; the view is supplementary. The user can re-run `/sb-view <slug>` directly to debug.
```

Do not auto-trigger `/sb-init`.

## Rules

- **Structure is mandatory.** A research run that doesn't produce all four files is incomplete — surface the gap, don't pretend it's done.
- **Cite everything in INSIGHTS.** Insights without source refs are opinions and have no place in a research artifact downstream stages will trust.
- **Don't synthesize new research findings in this orchestrator.** The deep-research skill does the synthesis; you do the filing.
- **Tight scope.** If the user pastes 30 PDFs and 12 repos in one invocation, ask whether to split into multiple topic-slugs. Cramming too much into one topic hurts INSIGHTS quality and downstream context efficiency.
- **Respect existing topics on augmentation.** Never delete source notes when augmenting; only add or update.

## Handling escalations

If `deep-research` returns blocked (sources unreachable, conflicting claims unresolvable, scope too large), surface to user:

```
QUESTION: <one sentence>
CONTEXT: <what the research found and what's stuck>
OPTIONS: A: ... / B: ...
RECOMMENDATION: <pick + rationale>
```

Wait for the user. Log non-trivial resolutions to `docs/decisions/<date>-<slug>.md` so future research and planning sees the precedent.
