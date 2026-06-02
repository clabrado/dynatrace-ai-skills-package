---
name: dt-rum-journey
description: >
  User Journey Funnel Diff (Tier 2) — given a RUM application, an ordered set of journey
  steps (UserActions or page URL patterns), and two time windows (or a deploy ID), compute
  per-step conversion rates, identify the largest drop-off ("leak"), find the most-frequent
  next action users took instead of progressing, and correlate those drop-offs to backend
  failures during the same window. Output is an executive MD + PDF report with funnel
  diagram, per-step delta table, leak analysis, and backend attribution.
---

# dt-rum-journey — User Journey Funnel Diff

Agentic Tier-2 skill. Accepts a RUM app, an ordered step list, and two time windows; runs
three parallel workers (funnel counts, leak detection, backend attribution); synthesizes
per-step deltas; passes the funnel diff to Davis CoPilot for cause-class classification;
emits an executive MD + ASCII-PDF report.

## Usage

```
/dt-rum-journey APP:"checkout-web" STEPS:"View Cart>Begin Checkout>Enter Payment>Place Order" \
                BASELINE:"2026-05-20T14:00Z..2026-05-20T18:00Z" \
                COMPARE:"2026-05-27T14:00Z..2026-05-27T18:00Z"

/dt-rum-journey APP:"shop-mobile" STEPS:"/products>/cart>/checkout>/confirmation" \
                --vs-deploy DEPLOY-7C3A8B91

/dt-rum-journey APP:"signup-web" STEPS:"/signup>/verify-email>/welcome" \
                BASELINE:"2026-05-21T00:00Z..2026-05-22T00:00Z" \
                COMPARE:"2026-05-27T00:00Z..2026-05-28T00:00Z" -clean
```

## Arguments

| Arg / flag | Description |
|---|---|
| `APP:"<rum-app>"` | RUM application `frontend.name`. Required. |
| `STEPS:"step1>step2>...>stepN"` | 2–7 ordered steps separated by `>`. URL paths start with `/`; otherwise UserAction names (`interaction.name`). |
| `BASELINE:"<iso>..<iso>"` | Pre-incident/pre-change window. |
| `COMPARE:"<iso>..<iso>"` | Post-incident/post-change window. |
| `--vs-deploy DEPLOY-<id>` | Alternative: derives BASELINE = 4h before deploy, COMPARE = 4h after deploy. |
| `-clean` | Sanitize URLs, page names, frontend names, user tags per dt-rca Phase 1.14. |

## STEPS Parsing Rule

| Leading char | Match rule | DQL predicate |
|---|---|---|
| `/` | URL path match | `page.url.path == "<step>"` on `has_page_summary == true` events |
| any other | UserAction name match | `interaction.name == "<step>"` on `has_user_action == true` events |

Mixed step lists are allowed. **Validated 2026-05-28:** `user_action.name` has zero rows on this tenant family; `interaction.name` is canonical.

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace RUM-instrumented frontend (`user.events` with real-user traffic)
- dynatrace-for-ai skills: `dt-obs-frontends`, `dt-obs-tracing`
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

| Worker | Reference files (read ONCE at worker start) |
|---|---|
| W-funnel | `dt-obs-frontends/SKILL.md` + `references/UserAction.md` + `references/PageViewAnalysis.md` + `references/user-sessions.md` |
| W-dropoff | `dt-obs-frontends/references/NavigationPatterns.md` + `references/UserAction.md` |
| W-attribution | `dt-obs-tracing/references/request-attributes.md` + `references/entity-lookups.md` |

## RUM DATA MODEL (NON-NEGOTIABLE)

- RUM events: `fetch user.events` — filter by `frontend.name == "{APP}"`
- Session key: **`dt.rum.session.id`** (with dot, NOT `dt.rum.session_id`)
- **ALWAYS filter `dt.rum.user_type == "real_user"`** — synthetic traffic distorts funnels
- `user.sessions` lookback: extend `from:` by at least 8h vs event window (sessions can last 8h+)
- Backend bridge: spans carry `dt.rum.session.id` and `dt.rum.trace_id` for RUM→backend join

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0b: Bootstrap (Sequential, 2-3 queries)

**Q1:** Sanity-check APP frontend exists:
```dql
fetch user.events, from:now()-24h
| filter frontend.name == "{APP}" and dt.rum.user_type == "real_user"
| summarize event_count = count(), page_summaries = countIf(characteristics.has_page_summary == true),
            user_actions = countIf(characteristics.has_user_action == true)
| limit 1
```
0 events → exit with "Frontend not found".

**Q2 (if --vs-deploy):** Resolve deploy timestamp, derive BASELINE/COMPARE with 15-min buffer around the deploy boundary.

**Q3 (optional, if action steps):** Verify `interaction.name` values match:
```dql
fetch user.events, from:toTimestamp("{BASELINE_FROM}"), to:toTimestamp("{COMPARE_TO}")
| filter frontend.name == "{APP}" and dt.rum.user_type == "real_user"
| filter characteristics.has_user_action == true
| summarize action_count = count(), by: {interaction.name}
| filter in(interaction.name, array({comma-separated quoted action steps}))
```

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

**W-funnel:** Per-step distinct session count for BASELINE and COMPARE windows:
```dql
fetch user.events, from:toTimestamp("{W.from}"), to:toTimestamp("{W.to}")
| filter frontend.name == "{APP}" and dt.rum.user_type == "real_user"
| filter {step_predicate_S_i}   -- page.url.path == "..." OR interaction.name == "..."
| summarize sessions_at_step = countDistinct(dt.rum.session.id)
```

Compute: `step_over_step_conversion[i] = sessions[i] / sessions[i-1]`, `delta_pp = compare_conv - baseline_conv`.

**W-dropoff:** For each step with `delta_pp ≤ -5`:
1. Next action for leakers (sessions at step i NOT at step i+1)
2. Session-end reason for leakers (`user.sessions` with 8h extended lookback)
3. Interrupted/timed-out user actions at leak step

**W-attribution:** For leak step — backend errors correlated to leaker sessions:
1. Failed spans for leaker sessions (sessions at S_i but not S_next)
2. Same query for progressor sessions (S_i AND S_next)
3. RUM-side errors at leak step (`characteristics.has_error == true`)

Differential rule: `leaker_fail_rate / progressor_fail_rate > 2.0` → `backend_attributed`.

### Phase 1.5: Absence Gate

For "step never reached" claims: run unfiltered `user.events` by `interaction.name` or `page.url.path` to confirm.

### Phase 2: Per-Step Delta Synthesis

```
For each step i in 1..N:
  sov[i] = sessions[i] / sessions[i-1]
  delta_pp[i] = (sov_c[i] - sov_b[i]) * 100

Classify:
  ≥ +5 pp  → "improved"
  |x| < 5  → "stable"
  ≤ -5 pp  → "leak"
  ≤ -15 pp → "severe leak"

biggest_leak = argmin_i(delta_pp[i])
```

### Phase 3: Davis CoPilot Synthesis

Pass funnel deltas + attributions (NOT raw rows) to Davis. Ask for: cause class (UX regression / backend regression / abandonment / navigation redirect), confidence, next investigation step.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 + H2 subtitle with APP, N steps, windows
2. Header + STEPS parsing rule banner
3. Executive Summary (Headline, Funnel Delta at a Glance, Cause Class, Immediate Action)
4. Funnel Diagram (Mermaid graph with leak step highlighted red)
5. Per-Step Delta Table
6. Leak Analysis (top next-actions, end-reason breakdown, complete-reason at step)
7. Backend Attribution (overall verdict + differential table + RUM-side errors + span failure sample)
8. Recommended Investigation
9. Appendix A: Methodology (STEPS rule, real_user filter, session key, conversion definition, leak threshold, backend differential rule)
10. *End of Report*

**Filenames:**
```
Normal: Funnel_{APP}_{BASELINE_DATE}_vs_{COMPARE_DATE}_Report.md / .pdf
Clean:  Funnel_{APP}_{BASELINE_DATE}_vs_{COMPARE_DATE}_Report_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **ALWAYS filter `dt.rum.user_type == "real_user"`** — state this in Methodology appendix.
2. **`dt.rum.session.id` with dot** — never `dt.rum.session_id` (zero rows).
3. **Step-over-step conversion per row** — `sov[i] = sessions[i] / sessions[i-1]`. End-to-end only in Executive Summary.
4. **Leak threshold = 5 pp** — below threshold = statistical noise. Severe = 15 pp.
5. **Backend attribution requires differential** — `leaker_rate / progressor_rate > 2.0` counts as attributed. Absolute counts alone are meaningless.
6. **STEPS parsing rule published in report** — user must be able to verify which match rule was applied.
7. **2–7 steps only** — fewer = not a funnel; more = split into sub-funnels.
8. **Sessions lookback +8h** for `fetch user.sessions` queries.
9. **UserAction names on `interaction.name`** — `user_action.name` has zero rows on most tenant families.

## Real-World Use Cases

1. **Cart abandonment investigation** — pre/post promo banner change; surfaces if conversion broke at a specific step.
2. **Onboarding funnel regression** — new signup flow; identify which step lost users WoW.
3. **Marketing campaign attribution** — campaign drove traffic; shows which step new users actually exited at.
4. **Deploy-induced UX regression** — `--vs-deploy` ties a specific release to a funnel break.
5. **A/B variant analysis** — compare same funnel between two cohorts to validate variant impact.
