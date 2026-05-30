---
description: Domain research stage upstream of /sb-init. Thin orchestration over the built-in deep-research skill that enforces a 3-tier output structure (INSIGHTS, BRIEF, source notes) so downstream stages can consume research without dumping it into context.
argument-hint: "<topic-slug> [inputs: PDF paths, repo URLs, freeform question — quote multi-token args]"
role: orchestrator
---

# Research (domain grounding)

You are the **orchestrator**. Domain research often belongs *before* product inception — building a learning app benefits from grounding in learning science before strategy questions are even framed. `/sb-research` runs that research and structures the output so `/sb-init` (and later `/sb-spec`, `/sb-plan`) can consume it without loading the entire corpus into conversation context.

This command is a **thin wrapper over the built-in `deep-research` skill.** All actual research, web fetching, and source-cross-checking happens there. This command's job is structure: enforce the output shape, write to the right place, append to the index.

Args: **$ARGUMENTS** — first token is `<topic-slug>` (lowercase kebab-case, 3-40 chars). Rest is the research question + any input pointers (file paths, URLs, repo locations).

## Why this exists

Without structured output, research artifacts get dumped into the next stage's context unfiltered — easily 50-300K tokens of paper summaries and repo evaluations. The intake-author then either ignores them (waste) or chokes on them (degradation). With structured output, downstream stages read 500 words of INSIGHTS, reference the BRIEF on demand, and only crack open source notes when something specific is challenged. Same intellectual leverage, 10× the context efficiency.

## Pre-flight

1. Parse `<topic-slug>`. Validate format: lowercase kebab-case. If invalid or missing, ask the user for a slug (e.g., `learning-science`, `competitor-landscape`, `motivation-by-age`).
2. Check `docs/research/<topic-slug>/` exists. If yes: "Topic exists. Augment, overwrite, or pick a new slug?" Augmenting means a new round adds sources and updates INSIGHTS/BRIEF without losing prior source notes.
3. Identify input sources from `$ARGUMENTS`:
   - PDF paths (anything ending `.pdf`) — read natively
   - URLs (http/https) — fetched via deep-research
   - Local file paths (e.g., requirement briefs) — read directly
   - Repo paths (local) or repo URLs — evaluated as competitor/prior-art artifacts
   - Freeform text — the research question itself

## Procedure

### Step 1: Frame the research question

If the user didn't provide a tight research question, ask one short question to scope it. Examples:
- "What's the decisive question you want answered?" (single best question)
- "Which decisions does this research need to inform — strategy, architecture, or both?"

Don't run open-ended research without a question. Open-ended produces sprawl; the structured-output requirement won't save you from a bad scope.

### Step 2: Dispatch the deep-research skill

Invoke `deep-research` with:

```
Question: <the framed research question>
Inputs to read: <PDF paths, URLs, repo locations the user provided>
Output requirement: structured per /sb-research convention — return a JSON-like packet with:
  - insights:    array of structured insight objects (see schema below)
  - brief:       1-2 page exec summary, markdown
  - sources:     array of {title, type, url_or_path, key_claims, our_notes}
  - confidence:  {high: [...], contested: [...], speculative: [...]}
Context budget: prioritize the user's named inputs (PDFs, repos). Web search is for filling gaps, not replacing the named corpus.
```

The deep-research skill does the actual work — fan-out searches, source-by-source reading, adversarial verification. This command's value is the output discipline.

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
- 300-700 words total. If you can't fit it, the research wasn't synthesized — fix that, don't expand the file.
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

1-2 pages. Downstream stages don't load this by default — they reference it when INSIGHTS surfaces a question that needs more.

#### `docs/research/<topic-slug>/SOURCES.md` — the source index

```markdown
# Sources: <topic>

| Ref | Title | Type | Location | Date | Read fully? |
|---|---|---|---|---|---|
| S1 | <title> | paper / repo / article / docs / interview | `sources/s1-<slug>.md` | <year> | ✅/⚠️ |
| S2 | ... | ... | <url or path> | ... | ... |
```

#### `docs/research/<topic-slug>/sources/<sN-slug>.md` — per-source notes

One file per source. Format:

```markdown
# [S<N>] <source title>

**Type:** paper / repo / article / docs / interview
**Original location:** <url or path>
**Read on:** <date>

## What it claims
<bulleted, specific>

## Quoted passages we relied on
> "<quote>" — page/section ref

## Our evaluation
<one paragraph: is this a strong source for this question? Where do we trust it, where do we hedge?>

## Cross-references
- Agrees with: [S3]
- Disagrees with: [S5] on <point>
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

INSIGHTS.md  (<word count>): docs/research/<topic-slug>/INSIGHTS.md
BRIEF.md     (<word count>): docs/research/<topic-slug>/BRIEF.md
SOURCES.md   (<n> sources):  docs/research/<topic-slug>/SOURCES.md
sources/     (<n> notes):    docs/research/<topic-slug>/sources/
INDEX.md updated.

Top 3 decisions implied:
1. <heading from INSIGHTS>
2. ...
3. ...

Open questions you'll need to answer in /sb-init:
- <list>

Next: /sb-research <next topic> if more research is needed, or /sb-init to start product inception with these insights grounded.
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
