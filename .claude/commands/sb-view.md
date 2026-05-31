---
description: Render /sb-research artifacts into a beautiful, navigable HTML site at .local/research-views/. No-arg renders the cross-topic dashboard and refreshes every topic's view. With a topic slug, renders that topic + refreshes the dashboard. Sized for 30-60 minute total human consumption.
argument-hint: "[<topic-slug>] [--open]"
role: orchestrator
---

# View (HTML rendering of research)

You are the **orchestrator**. The research MD files under `docs/research/` are agent-consumption artifacts (load-bearing for `intake-author`, the planner, the auditor). `/sb-view` produces a parallel **human-consumption artifact** — a self-contained HTML site at `.local/research-views/` — that you can browse, scan, and drill into in 30-60 minutes.

The HTML site is **not committed**. `.local/` is gitignored. Re-run `/sb-view` anytime to regenerate.

Args: **$ARGUMENTS** — optional first token is `<topic-slug>` (omit for cross-topic dashboard). Optional `--open` to launch the site in the default browser (macOS only via `open`).

## Pre-flight

1. Parse args: optional `<topic-slug>`, optional `--open` flag.
2. Verify `docs/research/INDEX.md` exists. If not, refuse: "No research yet. Run /sb-research <topic-slug> first."
3. **Ensure `.gitignore` excludes `.local/`.** If `.gitignore` exists and lacks `.local/` (or `.local`), append it with a comment:
   ```
   # /sb-view generates HTML research views here; not committed
   .local/
   ```
   If `.gitignore` doesn't exist, create it with the same content. **Do not** modify other entries.
4. Create `.local/research-views/` if it doesn't exist.
5. Determine project name for the dashboard hero: read `CLAUDE.md` for a `# {name}` H1 if present, otherwise fall back to the repo's directory name.

## Procedure

### Case A: no slug provided (dashboard mode)

The default invocation. Renders every topic + the cross-topic dashboard.

1. **Enumerate topics.** Glob `docs/research/*/INSIGHTS.md`. Build the list.
2. **Render each topic in parallel.** For each topic slug, dispatch `research-renderer` with `Mode: topic`. Use parallel agent calls — these are independent. If there are more than 4 topics, batch into groups of 4 to respect concurrency limits.
3. **Render the dashboard.** After all topic renders return DONE, dispatch `research-renderer` with `Mode: dashboard`. (Must run after topic mode because the dashboard's reading-time estimates aggregate from per-topic counts.)
4. **Surface result** (see SURFACE format below).

If any topic-mode renderer escalates, surface the escalation but continue rendering the remaining topics + the dashboard. Partial views are better than nothing.

### Case B: slug provided (single-topic mode)

Targeted re-render of one topic. Faster than the full dashboard pass.

1. Verify `docs/research/<slug>/INSIGHTS.md` exists. If not, suggest: "Topic `<slug>` not found. Run /sb-research <slug> first, or list existing topics: <list>." Stop.
2. Dispatch `research-renderer` with `Mode: topic`, slug as input.
3. **Always also refresh the dashboard.** Single topic re-render means the dashboard's "last updated" and reading-time aggregates need a refresh. Dispatch `research-renderer` with `Mode: dashboard` after the topic render returns. (Cheap — only the index page changes.)
4. **Surface result.**

## SURFACE format

```
─── /sb-view complete ─────────────────────────────────

Mode:           <dashboard | topic: <slug>>
Topics:         <n> rendered
Files written:  <n>
Total time:     <scan> min scan · <deep> min deep

Dashboard:      file:///<absolute path>/.local/research-views/index.html

Topic pages:
  <slug-1>     scan <n> · deep <n> min   file://.../index.html
  <slug-2>     scan <n> · deep <n> min   file://.../index.html
  ...

To open:        open .local/research-views/index.html      (macOS)
                xdg-open .local/research-views/index.html  (Linux)
                start .local/research-views/index.html     (Windows)
```

If `--open` was passed AND running on macOS (detect via `uname` Bash call returning `Darwin`), run `open .local/research-views/index.html` after surfacing. Otherwise print the open commands and let the user invoke.

## Idempotency and freshness

- Re-running `/sb-view` with no arg fully refreshes everything. Safe to spam.
- Re-running `/sb-view <slug>` is the fast path for "I just re-ran research on one topic; show me the updated view." It also refreshes the dashboard.
- The renderer writes `_assets/style.css` only when missing or stale. Repeated runs don't churn the design file.
- All output is under `.local/` — no risk of accidental commit.

## When to use which command

| You did | Use this |
|---|---|
| Just ran `/sb-research <slug>` and want to see it | `/sb-research <slug> --view` (composes — runs research then sb-view automatically), or `/sb-view <slug>` after |
| Want to browse all research before /sb-init | `/sb-view` (no args) |
| Re-rendered after improving the design system | `/sb-view` (no args) |
| Want to share research with a teammate | `/sb-view` (no args), then send the user the `.local/research-views/` directory or zip it |

## Why `.local/` (not `tmp/`, not `~/Documents/`)

- **In the repo workspace** — discoverable, doesn't require remembering an external path.
- **Gitignored** — never accidentally committed.
- **Persists across sessions** — unlike `/tmp/`, you can leave the browser open across days.
- **Same workspace as the source** — relative file:// paths work cleanly; the user can drag the folder elsewhere without breaking the site.

If the user explicitly wants the output elsewhere, that's a one-line override in their `.gitignore` and a different output dir in this command. Not exposing as a flag in v1.

## Rules

- **Don't render if there's no research.** Refuse gracefully, point to `/sb-research`.
- **Never modify `.gitignore` beyond appending `.local/`.** If other entries exist, leave them. If `.local/` is already present (any form), don't duplicate.
- **Never delete files in `.local/research-views/`.** The renderer overwrites in place. Orphaned files from deleted topics will accumulate; mention this at end-of-run if the topic count went down ("Note: .local/research-views/ contains topics no longer in docs/research/. Run `rm -rf .local/research-views && /sb-view` to clean.").
- **Surface, don't auto-open by default.** `--open` is opt-in. The user's browser might already be busy.

## Handling escalations from research-renderer

If the renderer escalates (missing source file, malformed INSIGHTS, etc.):
- Continue rendering remaining topics + dashboard.
- Surface the escalation with: `[topic-slug]` failed → reason. Suggested fix: re-run `/sb-research <slug>` to regenerate the source MD.
- Mark the failed topic in the dashboard with a small "render failed" notice and a link to the source MD directory so the user can investigate.
