---
name: product-concerns
description: Use when auditing a plan or feature against product-fit concerns (scale, security, accessibility, reliability, observability). Provides concrete checklists per dimension so audits aren't vibes. Trigger when running /sb-plan's concerns audit, or whenever a user asks "did we think about X" where X is one of these dimensions.
---

# Product concerns checklists

Five dimensions, one checklist each. Walk every category for each dimension. For each item, the answer is one of:

- ✅ **Handled** — point to where in plan/code
- ⏳ **Deferred** — with rationale (good)
- ❌ **Missed** — blocking finding
- N/A — with reason

The "N/A with reason" matters. "N/A because we don't care" is not acceptable; "N/A because this feature is read-only" is.

---

## Scale

How does this behave under load that's 10x, 100x, 1000x today's?

- **Query patterns** — Does this read pattern still work at 10x rows? Is there an index? Is the query bound by something other than total table size (date range, user, etc)?
- **Write amplification** — Does one user action cause N writes? What's N at scale?
- **Memory footprint** — Does this hold collections in memory that grow with usage?
- **Network round-trips** — Is there an N+1 lurking? Batch where possible.
- **Background jobs** — If async, does the queue depth grow faster than workers can drain at peak?
- **Hot keys / cardinality** — Any IDs that concentrate traffic (popular content, single tenant)? Sharding answer?
- **Caching** — Is there a cache? Cache invalidation strategy? Cache stampede protection?
- **Pagination** — User-facing lists paginated? API responses bounded?

**Single most-missed item:** "we'll handle it at scale" without a defined threshold. Push back: at what number of users / rows / requests do we revisit this?

---

## Security

What can an attacker do? What can an authenticated user do that they shouldn't?

- **Authentication** — Required where it should be? Session/token expiration?
- **Authorization (the bigger miss)** — *who* can do *what* to *which* resource? Specifically:
  - Can user A read user B's data via direct ID? (IDOR)
  - Can user A modify resources they shouldn't own?
  - Are admin endpoints actually checked, not just routed differently?
- **Injection surfaces** — SQL, shell, template, path traversal, regex. Parameterized? Escaped?
- **Sensitive data** — Secrets hardcoded? PII in logs? Sensitive fields in API responses that shouldn't be there?
- **Error messages** — Do they leak stack traces, table names, internal IDs to unauthenticated users?
- **CSRF / clickjacking / CORS** — If browser-facing, are these set right?
- **Rate limiting / abuse** — Endpoints that mutate or are expensive: bounded per user/IP?
- **Cryptography** — Custom crypto present? (Almost always wrong.) Timing-safe comparisons for tokens?
- **Dependencies** — New packages added; any known CVEs? Supply chain risk?
- **Idempotency on mutations** — Webhooks, payments, notifications: replay-safe?

**Single most-missed item:** authorization (who can do what) vs authentication (who is this). Conflating them is the most common audit miss.

---

## Accessibility

Can everyone use this?

- **Keyboard navigation** — Every interactive element reachable by keyboard? Focus visible? Logical tab order?
- **Screen readers** — Semantic HTML? Labels on inputs? ARIA only where semantic HTML can't express the role?
- **Color contrast** — WCAG AA minimum (4.5:1 body, 3:1 UI components)?
- **Color-only signals** — Status conveyed by color also conveyed by icon or text?
- **Motion** — `prefers-reduced-motion` respected? Critical info not conveyed only via motion?
- **Touch targets** — 44x44pt minimum on touch? (Desktop power-tools can go smaller if keyboard exists.)
- **Forms** — Errors associated with their inputs? Required fields marked beyond just color?
- **Language and locale** — `lang` attribute set? Text rendered in user's locale where applicable?
- **Zoom and reflow** — Does the UI work at 200% zoom without horizontal scroll?
- **Time limits** — Any UI that times out? User can extend?

**Single most-missed item:** focus visibility. Defaults strip it, custom styles often forget to restore it, keyboard users get lost.

---

## Reliability

What happens when something breaks?

- **Failure modes enumerated** — Network drops, downstream timeout, partial writes, queue full, database unavailable
- **Retry strategy** — Retries with backoff? Cap on attempts? Idempotency required for retry-safe?
- **Timeouts** — All external calls have explicit timeouts? Not just default 30s?
- **Circuit breakers** — Failing dependency doesn't take down the whole system?
- **Graceful degradation** — If non-critical dependency fails, does the core flow still work?
- **Data integrity** — Multi-step writes wrapped in transactions where needed? No half-applied state?
- **Rollback** — If we ship this and it's wrong, how do we undo? Schema migrations reversible? Feature flag in place?
- **Backup / recovery** — If data is deleted incorrectly, can we get it back?
- **Capacity planning** — Do we know what "overloaded" looks like before we hit it?

**Single most-missed item:** timeouts on external calls. Defaults are usually too long, and a stuck call propagates fast.

---

## Observability

How do we know it's working in production?

**Most important check: metric instrumentation traceability.** Before walking the generic list below, pull metrics from `docs/STRATEGY.md#metrics` (project-level) and the plan's `## Success metrics` section (feature-level). For each metric, find where in the plan it's instrumented. A metric without an instrumentation path in the plan's Steps is a **blocking finding** — you cannot measure what you cannot see.

Build a traceability table as part of this dimension's output:

| Metric | Source | Instrumented where? | Status |
|---|---|---|---|
| Weekly active users | STRATEGY#metrics | Plan §steps step 4 (event `feature.used` to analytics) | ✅ |
| P95 latency on new endpoint | plan §metrics | NOT FOUND in plan | ❌ blocking |
| Webhook dedupe hits | plan §metrics | Plan §steps step 7 (counter `webhook_dedupe_hits`) | ✅ |
| Adoption rate | STRATEGY#metrics | NOT FOUND | ❌ blocking |

If a project-level metric is genuinely not relevant to this feature, mark it N/A with reason ("this feature is a backend job, not user-facing — adoption doesn't apply"). N/A is acceptable; silent omission is not.

**Generic observability checks** (after the metric traceability table):

- **Logs** — Key user actions and system events logged? Structured logs (not just text)? Correlation IDs threaded through requests?
- **Metrics** — Request rate, error rate, latency p50/p95/p99 captured? Business metrics (signups, conversions, retention)?
- **Traces** — For distributed flows: end-to-end traces with spans across services?
- **Alerts** — What pages on-call? What's the threshold? Is it tunable without code change?
- **Dashboards** — Can someone unfamiliar with the code answer "is X healthy right now" in 30 seconds?
- **Error reporting** — Exceptions captured to a service (Sentry/Honeycomb/etc.) with stack traces and context?
- **Audit trail** — For sensitive actions (auth changes, payments, deletions): logged for forensics?
- **User-facing status** — When something's broken, do users see useful errors, not opaque 500s?
- **Feature flags as telemetry** — Can we tell what % of users are on the new path?

**Single most-missed item:** error rate by error class. "Errors went up" is not actionable; "errors of type X spiked" is.

**Why the metric traceability check is the load-bearing one:** the rest of the observability checklist is generic — every project should have logs, alerts, dashboards. The metric traceability is project-specific: it links the plan to the strategy. If the named metrics aren't instrumented, `/sb-eval` becomes a vibes check 30 days later, and the whole compound learning loop breaks down.

---

## How to use this skill in audits

The `concerns-auditor` agent loads this skill on dispatch. It walks each dimension's checklist against the plan, producing one finding row per item. Output format:

```markdown
## <Dimension>

| Category | Status | Detail |
|---|---|---|
| Query patterns | ✅ | Indexed on user_id (plan §affected-files) |
| Pagination | ❌ | `/list-todos` endpoint returns unbounded array |
| Caching | N/A | Read-once-per-day data, no cache needed |
```

A dimension passes if zero ❌ items and all ⏳ items have rationale.

## When to skip a dimension

If a feature genuinely doesn't touch a dimension — e.g., a backend-only feature with no UI doesn't need accessibility — the auditor returns the entire dimension as "N/A: <reason>". This is acceptable; forcing irrelevant audits trains the team to skim them.
