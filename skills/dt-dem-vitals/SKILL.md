---
name: dt-dem-vitals
description: >
  Web Vitals Regression Brief — a focused before/after comparison of Core Web Vitals
  (INP, LCP, CLS, FCP, TTFB) for a Dynatrace RUM application across two windows, or
  automatically derived around a deployment. Identifies regressed pages with
  material deltas, attributes each regression to a class (frontend-rendered /
  backend-bound / network-bound), and correlates backend-bound regressions to
  XHR slowdowns and distributed traces. Produces a polished MD + PDF brief.
---

# dt-dem-vitals — Web Vitals Regression Brief

Agentic frontend regression skill. Diffs Core Web Vitals (INP, LCP, CLS, FCP, TTFB)
between a BASELINE window and a COMPARE window for a Dynatrace RUM application,
attributes regressions to frontend / backend / network, and emits a polished
Markdown + PDF brief.

## Usage

```
/dt-dem-vitals APP:"www-checkout" BASELINE:"2026-05-20T00:00Z..2026-05-21T00:00Z" COMPARE:"2026-05-27T00:00Z..2026-05-28T00:00Z"
/dt-dem-vitals APP:"www-checkout" --vs-deploy DEPLOY-83A21F
/dt-dem-vitals APP:"mobile-ios" BASELINE:"..." COMPARE:"..." -clean
```

## Arguments

| Arg / flag | Description |
|---|---|
| `APP:"<rum-app-id>"` | RUM application `frontend.name`. Required. |
| `BASELINE:"<iso>..<iso>"` | Explicit window (paired with COMPARE). Inclusive start, exclusive end. |
| `COMPARE:"<iso>..<iso>"` | Paired with BASELINE. |
| `--vs-deploy <DEPLOY-ID>` | Alternative to BASELINE/COMPARE. Sets BASELINE = 24h pre-deploy, COMPARE = 24h post-deploy. |
| `-clean` | Sanitize URLs, page paths, hostnames, RUM app names per dt-rca Phase 1.14. |
| `appendix=true` | Include full Appendix B (Glossary) + C (Capabilities). |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with RUM app emitting `dt.frontend.web.page.*` metrics
- dynatrace-for-ai skills: `dt-obs-frontends`, `dt-obs-tracing`
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-vitals | `dt-obs-frontends/references/WebVitals.md` (vital metric names + `timeseries percentile()` patterns) |
| W-regressions | Synthesizer worker — reads no reference; consumes W-vitals output |
| W-attribution | `dt-obs-frontends/references/RequestPerformance.md` + `RequestTimingAnalysis.md` + `TraceCorrelation.md` + `dt-obs-tracing/references/entity-lookups.md` + `request-attributes.md` |

## RUM DATA MODEL (NON-NEGOTIABLE — Tenant-Validated 2026-05-28)

**Web Vitals are METRICS, not events:**
- `fetch dt.rum.web.events` → `UNKNOWN_DATA_OBJECT` (does not exist)
- `fetch user.events | filter isNotNull(web_vitals.lcp)` → vital fields DO NOT exist on `user.events`
- **Correct:** `timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), by:{dt.entity.application}`

**Validated vital metric names:**

| Vital | Metric | Status |
|---|---|---|
| LCP | `dt.frontend.web.page.largest_contentful_paint` | data present |
| CLS | `dt.frontend.web.page.cumulative_layout_shift` | data present |
| INP | `dt.frontend.web.page.interaction_to_next_paint` | data present |
| FID | `dt.frontend.web.page.first_input_delay` | data present |
| FCP | `dt.frontend.web.page.first_contentful_paint` | metric exists; may be sparse |
| TTFB | `dt.frontend.web.page.time_to_first_byte` | metric exists; may be sparse |

**Inject windows via CLI flags — NEVER in DQL:**
```bash
DTCTL_TOKEN_STORAGE=file dtctl query \
  --default-timeframe-start "{WINDOW.from}" \
  --default-timeframe-end   "{WINDOW.to}" \
  'timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

**Backend attribution:** `user.events` + spans joined by `trace.id`. Filter `frontend.name == "{APP_NAME}"` on user.events. Use `getNodeName(dt.smartscape.service)` for service names on spans.

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0b: Context Bootstrap (Sequential, ≤4 queries)

**Q1:** Resolve APP entity ID:
```dql
fetch dt.entity.application
| filter entity.name == "{APP_NAME}" or contains(entity.name, "{APP_NAME}")
| fields id, entity.name | limit 5
```
0 rows → exit with "Application Not Found". Multiple rows → list and exit.

**Q1b:** Confirm metric data exists for entity (LCP over last 7d).
0 records → exit with "Application Not Reporting Vitals".

**Q2 (if `--vs-deploy`):** Resolve deploy event, derive BASELINE/COMPARE:
```
BASELINE = [DEPLOY_TIMESTAMP - 24h, DEPLOY_TIMESTAMP]
COMPARE  = [DEPLOY_TIMESTAMP, DEPLOY_TIMESTAMP + 24h]
```

**Q3:** Per-window sanity check (LCP intervals). If `intervals < 4`, set `LOW_TRAFFIC_WARNING = true`.

**Tenant dimension gap:** if metric isn't dimensioned by `dt.entity.application`, degrade gracefully to tenant-aggregate (never fail).

### Phase 1: Parallel Worker Dispatch

Dispatch W-vitals and W-attribution simultaneously. W-regressions dispatched AFTER W-vitals returns.

**W-vitals:** 12 queries in parallel (6 vitals × 2 windows). For each window and vital:
```bash
DTCTL_TOKEN_STORAGE=file dtctl query \
  --default-timeframe-start "{W.from}" --default-timeframe-end "{W.to}" \
  'timeseries <alias> = percentile(<metric>, 75), by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

Pool each window's sli array (p75 of non-null values → `baseline_value`/`compare_value`). Delta = compare − baseline.

**W-regressions (after W-vitals):** Apply material-delta thresholds:

| Vital | Trigger |
|---|---|
| INP | `compare ≥ baseline × 1.10` AND `compare - baseline ≥ 50ms` |
| LCP | `compare - baseline ≥ 200ms` |
| CLS | `compare - baseline ≥ 0.05` |
| FID | `compare - baseline ≥ 25ms` |
| FCP | `compare - baseline ≥ 200ms` |
| TTFB | `compare - baseline ≥ 100ms` |

Classify dominant vital (largest normalized delta = delta / threshold).

**W-attribution (parallel with W-vitals):** Identify top-5 candidate pages by p75 request duration in COMPARE window from `user.events`. For each: top XHRs, TTFB phase decomposition (DNS/connect/TLS/server/download), backend trace correlation via `trace.id` join.

### Phase 1.5: Absence Gate

For "no events at step X" claims: run unfiltered `user.events` confirmation by `interaction.name` or `page.url.path`.

### Phase 2: Regression Classification

For each regressed page, classify into ONE of (first match wins):
1. **`backend-bound`** — `backend_ratio ≥ 60%` OR TTFB server phase dominates
2. **`network-bound`** — TTFB regressed AND dominant phase is DNS/connect/TLS/download (not server)
3. **`frontend-rendered`** — default

### Phase 3: Davis CoPilot Synthesis

Pass classified regression list + attributions to Davis. Ask for: most likely driving page, owner team per page, single highest-leverage remediation step, cross-page patterns.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 + H2 subtitle with regression count + dominant class
2. Header + STEPS parsing rule banner
3. Executive Summary (Critical Findings, Impact Summary, Business Impact, Immediate Action, Regression Class pie)
4. Application-Level Vitals Delta table (all 6 vitals)
5. Per-Page Attribution (top-N candidate pages)
6. Backend Attribution Table (once, not per-page)
7. Recommended Owners table (once)
8. Davis Synthesis
9. Appendix A: Investigation Details + worker telemetry
10. Links + *End of Report*

**Filenames:**
```
Normal: VITALS_{APP_SLUG}_{DATE}.md / .pdf
Clean:  VITALS_{APP_SLUG}_{DATE}_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **Web Vitals are METRICS** — `timeseries` over `dt.frontend.web.page.*`. Never `fetch user.events` for vitals.
2. **Two-window discipline** — same DQL run twice via CLI flags; never inline `from:`/`to:` in DQL.
3. **Material-delta thresholds are gates** — below threshold = not regressed. Never invent new thresholds.
4. **Classification is single-class** — exactly one of frontend-rendered / backend-bound / network-bound per page.
5. **Backend attribution is trace-driven only** — never claim backend causation without `trace.id` join evidence.
6. **APP_NAME → APP_ENTITY_ID resolution required** — every vitals query filters by entity ID, never by name.
7. **Degrade gracefully** — sparse vitals → omit with note; absent metric dimension → tenant-aggregate scope.
8. **Filename = `VITALS_*`** — never dt-rca or dt-rcf naming.

## Real-World Use Cases

1. **Pre/post deploy comparison** — auto-window via `--vs-deploy` shows if new bundle shipped a Web Vitals regression.
2. **Black Friday traffic shift** — compare last week vs peak hour; surfaces whether load-driven INP degradation is real.
3. **A/B variant rollout analysis** — feature flag flip caused user complaints; skill quantifies vital shift.
4. **Geo-targeted regression** — `--with-geo` surfaces regression that hits APAC but not US (CDN edge issue).
5. **Customer DEM demo** — `-clean` mode produces polished regression artifact for SE customer call.

## Substrate Notes (Validated 2026-05-28)

- Vitals live in `dt.frontend.web.page.*` metric family, NOT in RUM events
- `dt.frontend.web.page.*` on demo tenant had `dt.entity.application == null` dimension gap — skill degrades to tenant-aggregate and documents this explicitly
- FCP and TTFB may be sparse; handled as omit-with-note, never a run failure
- `user.events` IS used for W-attribution (per-page XHR + trace.id correlation) — that path is valid
