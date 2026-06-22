---
name: dt-dem-vitals
description: >-
  Web Vitals Regression Brief — a focused before/after comparison of Core Web Vitals
  (INP, LCP, CLS, FCP, TTFB) for a Dynatrace RUM application across two windows, or
  automatically derived around a deployment. Identifies regressed pages with
  material deltas, attributes each regression to a class (frontend-rendered /
  backend-bound / network-bound), and correlates backend-bound regressions to
  XHR slowdowns and distributed traces. Composes dt-obs-frontends + dt-obs-tracing
  and produces a polished Markdown brief (PDF optional via `--pdf`). Trigger keywords: "Web Vitals regression",
  "INP regressed", "LCP got worse", "CLS spike", "RUM regression", "compare web vitals
  baseline", "vitals after deploy", "p75 INP delta", "page experience regression",
  "frontend slowdown after release".
---

# dt-dem-vitals — Web Vitals Regression Brief

Agentic frontend regression skill. Diffs Core Web Vitals (INP, LCP, CLS, FCP, TTFB)
between a BASELINE window and a COMPARE window for a Dynatrace RUM application,
attributes regressions to frontend / backend / network, and emits a polished
Markdown + PDF brief suitable for Slack-pasting to a frontend team or product owner.

This skill is the **DEM (Digital Experience Monitoring) sibling** of `/dt-rcf`. It
adopts the same phased substrate (Phase 0a parse, Phase 0-auth pre-warm, Phase 0b
context bootstrap, Phase 0c reference strategy, Phase 1 parallel worker dispatch in
ONE message, Phase 1.5 absence-gate, Phase 2 synthesis, Phase 3 Davis CoPilot,
Phase 4 report). It does NOT carry incident DQL — this is a baseline-vs-compare
diff, not a problem-anchored RCA.

---

## Usage

```
/dt-dem-vitals APP:"EasyTrade" BASELINE:2026-06-01T12:00:00Z COMPARE:2026-06-03T12:00:00Z
/dt-dem-vitals APP:"www-checkout" --vs-deploy DEPLOY-83A21F
/dt-dem-vitals APP:"mobile-ios" BASELINE:2026-05-20T00:00:00Z COMPARE:2026-05-27T00:00:00Z -clean
/dt-dem-vitals APP:"www-checkout" --vs-deploy DEPLOY-83A21F -clean
/dt-dem-vitals APP:"www-account" BASELINE:2026-05-20T14:00:00Z COMPARE:2026-05-27T14:00:00Z --window 4h
/dt-dem-vitals APP:"EasyTrade" BASELINE:2026-06-01T12:00:00Z COMPARE:2026-06-03T12:00:00Z --pdf
```

> **Markdown is the default deliverable.** PDF is opt-in via `--pdf` (or legacy
> `pdf=true`) and never blocks a run. **Window form:** `BASELINE:<iso>
> COMPARE:<iso>` are single instants; each window spans `[<point> .. <point> +
> WINDOW_LEN]` (default 24h, override with `--window`). The legacy `BASELINE:"a..b"`
> range form is still accepted.

## Arguments

- `APP:"<rum-app-id>"` — REQUIRED. The RUM application's `frontend.name` (web) or
  mobile application name. Use the exact name from the Dynatrace UI; the bootstrap
  resolves the entity ID.
- `BASELINE:<iso8601>` — single ISO-8601 instant; the START of the baseline window
  (PRIMARY form, DV-2). Paired with COMPARE. The window END is derived as
  `BASELINE + WINDOW_LEN` (default 24h). Legacy range form `BASELINE:"<iso>..<iso>"`
  is still accepted (used verbatim).
- `COMPARE:<iso8601>` — single ISO-8601 instant; the START of the compare window.
  Paired with BASELINE. Same window-derivation rule.
- `--window <dur>` — optional WINDOW_LEN override (e.g. `4h`, `48h`); default `24h`.
  Applies to both windows when the single-point pair form is used.
- `--vs-deploy <DEPLOY-ID>` — alternative to BASELINE/COMPARE. Bootstrap looks up the
  deployment event, sets BASELINE = the 24 hours preceding the event, COMPARE = the
  24 hours following the event.
- `-clean` — sanitize URLs, page paths, hostnames, RUM app names per the dt-rca
  Phase 1.14 rules adapted for RUM data (see Phase 1.14 below).
- `--pdf` — opt-in PDF generation (default OFF; Markdown is the canonical deliverable).
  Legacy alias `pdf=true`; `pdf=false` is a no-op.
- `appendix=true` — include Appendix B (Glossary) + Appendix C (Capabilities) in full;
  default OFF (saves ~100 lines).

> **Speed note:** run at **medium effort**. Higher effort multiplies subagent and
> writer latency without improving query quality; quality is set by which references
> the workers read and how tight the DQL scoping is.

## Sub-Skills Loaded Per Phase

dt-dem-vitals carries NO RUM DQL beyond the orchestrator's bootstrap. Each worker
reads ONE targeted reference file (its authority) at start, once per subagent, then
derives queries from it using the dt-dem-vitals scoping rules below. If a field or
metric changes upstream, only the reference needs fixing.

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-vitals | `~/.agents/skills/dt-obs-frontends/references/WebVitals.md` (vital metric names + `timeseries percentile()` patterns) |
| W-regressions | (Synthesizer worker — reads no reference. Consumes W-vitals output, applies regression thresholds, emits regressed-page list.) |
| W-attribution | `~/.agents/skills/dt-obs-frontends/references/RequestPerformance.md` + `RequestTimingAnalysis.md` + `TraceCorrelation.md` + `~/.agents/skills/dt-obs-tracing/references/entity-lookups.md` + `request-attributes.md` |

---

## Skill Registry

**Maintenance-time reference map** — the authoritative source for where each inline
query's DQL came from. Used when updating/re-validating the skill, and as a
per-query fallback if a field/metric name errors at run time. NOT read during
normal runs. **Never guess a path, metric, or field name — fix it here and
re-validate.**

**Re-validation (maintenance) — HARD GATE:** every inline DQL block in this skill
must be run live against the tenant before first production use. The block-level
`<!-- VALIDATE: ... -->` comment on each inline query is the marker. The
substrate was tenant-validated on 2026-05-28 (vitals metrics confirmed
present; `dt.rum.web.events` confirmed absent; `user.events` confirmed without
vital fields). Per-app data may still vary — re-run via
`DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'` (single quotes) before saving.
Common breakers: backticks (use aliased bins); `start_time` for `user.events`
and `spans`, `timestamp` for `logs`; metric vitals require
`dt.entity.application` filter (the resolved APP_ENTITY_ID), `user.events`
attribution queries use `frontend.name`; alias every `bin()`. For windowed
metric queries, inject windows via `--default-timeframe-start/--default-timeframe-end`
flags — never put `from:`/`to:` in the DQL.

### DQL Authority

| File | Purpose | When to read |
|------|---------|--------------|
| `~/.agents/skills/dtctl/references/DQL-reference.md` | Core DQL syntax, filter patterns, aggregation | Orchestrator Phase 0c |
| `~/.agents/skills/dt-dql-essentials/references/dql/dql-functions-smartscape.md` | smartscapeNodes, getNodeName, getNodeField exact signatures | Orchestrator Phase 0c (only if --vs-deploy resolves to a deployment event needing entity context) |

### W-vitals — dt-obs-frontends

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-frontends/SKILL.md` | Always — W-vitals |
| `~/.agents/skills/dt-obs-frontends/references/WebVitals.md` | Always — vital metric names + `timeseries percentile()` patterns |
| `~/.agents/skills/dt-obs-frontends/references/AdvancedPerformance.md` | Conditional — only if `--with-geo` AND probe succeeds (geo dimension on metrics is v2-gated) |

### W-attribution — dt-obs-frontends + dt-obs-tracing

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-frontends/references/RequestPerformance.md` | Always — `dt.frontend.request.duration` timeseries + browser/device split |
| `~/.agents/skills/dt-obs-frontends/references/RequestTimingAnalysis.md` | Always — TTFB decomposition (DNS / connect / TLS / server / download) |
| `~/.agents/skills/dt-obs-frontends/references/TraceCorrelation.md` | Always — trace.id linkage from `user.events` to spans (backend attribution) |
| `~/.agents/skills/dt-obs-tracing/references/entity-lookups.md` | Always — `getNodeName(dt.smartscape.service)` for backend service names in correlation output |
| `~/.agents/skills/dt-obs-tracing/references/request-attributes.md` | Conditional — only if W-attribution finds custom `request_attribute.*` on the joined spans (rare for RUM-driven traces; useful for B2B portals with custom client headers) |

---

**Sanitization, URL format, Mermaid/ASCII diagram rules, and PDF generation:**
These are identical to `/dt-rca` Phase 1.14, Phase 3, Phase 4, "PDF DIAGRAM RULES",
"MERMAID SYNTAX RULES", and "DIAGRAM SIZING RULES" — read `~/.claude/skills/dt-rca/SKILL.md`
for the canonical rules. The dt-dem-vitals adaptation of Phase 1.14 (RUM-specific
sanitization targets) is documented in this file.

---

## RUM DATA MODEL (NON-NEGOTIABLE — TENANT-VALIDATED)

dt-dem-vitals queries Core Web Vitals as **METRICS** via `timeseries` over the
`dt.frontend.web.page.*` metric family, grouped by `dt.entity.application`.
Backend attribution uses `user.events` + spans joined by `trace.id` (the
`user.events` data object exists and is used for the attribution path only).

> **CRITICAL — DO NOT do this (validated 2026-05-28):**
> - `fetch dt.rum.web.events` → `UNKNOWN_DATA_OBJECT` (does not exist).
> - `fetch user.events | filter isNotNull(web_vitals.largest_contentful_paint)` →
>   `user.events` exists (~1.1M records/hr on tenant) but does NOT carry
>   LCP/INP/CLS/FCP/TTFB as event fields. Vitals live in the metric family below.
>
> If a vitals query fails with `UNKNOWN_DATA_OBJECT` or `FIELD_DOES_NOT_EXIST`,
> the worker MUST switch to the `timeseries` metric path — never fall back to
> "no data".

### Vital Metric Names (tenant-validated 2026-05-28)

| Vital | Metric | Tenant status |
|---|---|---|
| LCP | `dt.frontend.web.page.largest_contentful_paint` | data present — primary |
| CLS | `dt.frontend.web.page.cumulative_layout_shift` | data present — primary |
| INP | `dt.frontend.web.page.interaction_to_next_paint` | data present — primary |
| FID | `dt.frontend.web.page.first_input_delay` | data present — primary |
| FCP | `dt.frontend.web.page.first_contentful_paint` | metric exists, may be sparse |
| TTFB | `dt.frontend.web.page.time_to_first_byte` | metric exists, may be sparse |

**Sparse-vital handling:** if a vital's `timeseries` returns zero records for
BOTH windows, omit that vital from the regression table with a one-line note in
Appendix A (`"FCP omitted: no metric samples in either window"`). Do NOT fail
the run.

### Canonical Vitals Query Shape (tenant-validated)

```dql
timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75),
           interval:{VITALS_INTERVAL},
           by:{dt.entity.application}
```

> **DV-1 — `interval:` is MANDATORY on every windowed vitals timeseries.** When
> the window is injected via the CLI timeframe flags over a short (1–2d) span,
> the auto-interval misbehaves and the percentile timeseries returns **0
> records**. With an explicit `interval:1h` the same window returns ~24 clean
> intervals (proven live 2026-06-03 against `APP:"EasyTrade"`). Default
> `{VITALS_INTERVAL} = 1h`; for windows much longer than ~24h pick a coarser
> interval so each window yields ~24 buckets. Omitting `interval:` is a defect.

Time windows are injected via `dtctl query` CLI flags
`--default-timeframe-start "{WINDOW.from}" --default-timeframe-end "{WINDOW.to}"` —
**never** mutate the DQL with `from:`/`to:`. This keeps the same query string
re-usable for BASELINE and COMPARE and matches the substrate's preferred shape.

Result rows carry `dt.entity.application` (e.g., `APPLICATION-XXXXXXXXXXXX`)
and an `lcp` array of p75 values per time interval. The regression engine
pools the array (see "Regression delta computation" below).

### Application Entity Resolution

APP_NAME maps to a Smartscape application entity ID
(`APPLICATION-XXXXXXXXXXXXXXXX`). Phase 0b runs a `smartscapeNodes` lookup to
resolve the entity ID from the human-readable name, and every vitals query
filters by `dt.entity.application == "{APP_ENTITY_ID}"`.

### Page / Geo Dimensions — v1 LIMIT

The `dt.frontend.web.page.*` metrics on this tenant carry
`dt.entity.application` as their primary group-by dimension. **Per-page
granularity and `geo.country.iso_code` are NOT confirmed available as metric
dimensions in v1.** The orchestrator:

- v1 (default): produces APP-level vitals diff (one row per app, six vitals),
  not per-page. Per-page attribution still works through W-attribution because
  it uses `user.events` (which IS per-page).
- v2 (`--with-geo`): gated flag. Worker first probes
  `timeseries ... by:{geo.country.iso_code}` against the LCP metric for the
  BASELINE window; if it returns rows, geo split is enabled; otherwise the
  flag is rejected with a one-line warning and the run proceeds APP-level.

> **Tenant-validated 2026-05-28 dimension gap:** on this demo tenant,
> `dt.frontend.web.page.*` returns a SINGLE row with `dt.entity.application == null`
> when grouped by application — meaning the metric isn't dimensioned by app at all
> on this tenant family. Phase 0b's APP_ENTITY_ID probe (Q1b) MUST treat
> "no dimensioned rows" as a graceful no-op: the run proceeds as a tenant-aggregate
> Web Vitals delta (single global row covering all RUM traffic), and the report
> exec summary names this explicitly ("APP-level dimension unavailable on this
> tenant — output is tenant-aggregate"). The skill never fails on absent
> dimensions; it degrades to the broadest available scope and reports the scope.

### Page Identifier Selection (W-attribution only)

| App type | Identifier field | Notes |
|---|---|---|
| Web | `page.url.path` | On `user.events`. Strip query string + fragment; normalize trailing slash. |
| SPA (web) | `view.name` if `characteristics.has_view_summary` AND `navigation.type == "soft_navigation"` — else `page.url.path` | SPAs that haven't enabled view summaries fall back to `page.url.path`. |
| Mobile | `view.name` | iOS / Android view summaries on `user.events`. |

### Backend Attribution via trace.id

The frontend→backend join lives in `TraceCorrelation.md`. dt-dem-vitals uses the
"Slow Requests with Backend Traces" + "Backend Service Impact on Frontend" patterns
verbatim; **do not derive a custom join**.

```dql
-- Pattern (from TraceCorrelation.md, adapted to the regression page set):
fetch user.events, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
| filter frontend.name == "{APP_NAME}"
| filter characteristics.has_request == true
| filter page.url.path == "{REGRESSED_PAGE}"
| filter isNotNull(trace.id)
| filter duration > 1s
| fields trace.id, frontend_duration = duration, url.path
| join [
    fetch spans, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
    | summarize backend_duration = sum(duration),
                services = collectDistinct(getNodeName(dt.smartscape.service)),
                by: {trace.id}
  ], on: trace.id, fields: {backend_duration, services}
| fieldsAdd backend_ratio_pct = 100.0 * backend_duration / frontend_duration
| sort frontend_duration desc
| limit 50
```

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking — set up at the very start)

Maintain a per-run timing log so every run leaves diagnosable phase timing behind.
Set `LOG="VITALS_{APP_SLUG}_{DATE}.log"` (next to the report) and at each phase
boundary append ONE line, **backgrounded so it never gates execution**:

```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line summary>" >> "$LOG"; } &
```

Log: START, Phase 0b complete (App entity + windows resolved), Phase 1 dispatch
(per-worker one-liners), Phase 1.5 absence-gate result, Phase 2 regression count,
Phase 3 Davis result, report written, RUN COMPLETE. Phase 7 prints the log path.

---

### Phase 0a: Argument Parser

```
REQUIRED:
  APP:"<name>"                                  → APP_NAME

ONE-OF-THE-FOLLOWING (window source):
  BASELINE:<iso> COMPARE:<iso>                   → single-point pair (PRIMARY form, DV-2)
  --vs-deploy DEPLOY-XXXX                         → derive from deployment

OPTIONAL FLAGS:
  -clean         → CLEAN_MODE = true
  appendix=true  → FULL_APPENDIX = true (default false)
  --pdf          → PDF_MODE = true (default FALSE — Markdown is the canonical deliverable)
                   Legacy alias: pdf=true ⇒ --pdf; pdf=false ⇒ no-op (already the default)
  --window <dur> → WINDOW_LEN override (default 24h); each window spans
                   [<point> .. <point> + WINDOW_LEN]

WINDOW SOURCE — single-point pair form (DV-2, PRIMARY):
  BASELINE:<iso> and COMPARE:<iso> are single ISO-8601 instants (each the START
  of its window). Phase 0a derives each window's END as the next step:
    BASELINE.from = BASELINE_POINT          BASELINE.to = BASELINE_POINT + WINDOW_LEN
    COMPARE.from  = COMPARE_POINT           COMPARE.to  = COMPARE_POINT  + WINDOW_LEN
  Default WINDOW_LEN = 24h. (e.g. BASELINE:2026-06-01T12:00:00Z COMPARE:2026-06-03T12:00:00Z
  with default 24h ⇒ baseline 2026-06-01T12:00→06-02T12:00, compare 06-03T12:00→06-04T12:00.)

  LEGACY range form (accepted, deprecated): BASELINE:"<iso>..<iso>"
  COMPARE:"<iso>..<iso>". If a `..` range is supplied, use it verbatim as
  {from,to} and skip the WINDOW_LEN derivation. Prefer the single-point pair form;
  the range form is retained only for backward compatibility (note for doc-update pass).

VALIDATION:
  - APP_NAME REQUIRED — exit with usage if missing
  - Either (BASELINE+COMPARE single points or ranges) or --vs-deploy REQUIRED —
    exit with usage if neither
  - BASELINE/COMPARE points (or range endpoints) must be valid ISO-8601; warn if
    the derived windows overlap
  - --vs-deploy DEPLOY-ID format: alphanumeric, dash-prefixed
  - PDF_MODE defaults FALSE; only --pdf (or legacy pdf=true) sets it true
```

---

### Phase 0-auth: PRE-WARM the dtctl session (Orchestrator — proactive step 0)

**FIRST action of the whole run — before any data query and before any dispatch:**

```
dtctl auth refresh
```

This is a REFRESH, not a login — silent, no browser, uses the on-disk refresh token
(`DTCTL_TOKEN_STORAGE=file`, harness env, never the Keychain). It guarantees a
fresh access token up front, so the parallel workers all inherit one fresh token
instead of each triggering a concurrent refresh (a race).

- `dtctl auth refresh` succeeds → token fresh for the entire run; proceed to Phase 0b.
- It fails (no/expired refresh token — first-ever use, or weeks idle) → ONLY THEN run once:
  `dtctl auth login --plain --safety-level readonly` (read-only scopes; browser, one time).

**Workers never authenticate** — they inherit the pre-warmed on-disk token. If a
worker still returns `<gap: auth>`, the orchestrator refreshes once and re-dispatches
only that worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

---

### Phase 0b: Context Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** that all phase workers receive.

**MINIMAL BOOTSTRAP — at most 5 queries (1 probe + 4 bootstrap).** Bootstrap is
the orchestrator's only serial stretch, so keep it tiny. The goal is:

0. **Substrate Probe (Phase 0b.0)** — confirm the app entity resolves and the
   vitals metric family exists + populates with an explicit `interval:`, BEFORE
   any worker dispatch.
1. Resolve the RUM application's Smartscape entity ID from APP_NAME.
2. Validate the entity is producing vitals metric data.
3. If `--vs-deploy`, look up the deployment event to derive BASELINE / COMPARE.
4. Sanity-check window traffic via the LCP metric.

#### Phase 0b.0 — Substrate Probe (runs ONCE before the bootstrap queries below)

**Why:** the orchestrator's bootstrap formerly assumed (a) the application
resolves and (b) the vitals metric returns rows over the windowed flags. Both
assumptions drifted in test: a windowed `timeseries percentile(...)` returns
**0 records** without an explicit `interval:` (DV-1), and the app may resolve via
`fetch dt.entity.application` (PRIMARY) or — only if that errors/empties — the
MCP `find_entity_by_name` fallback (DV-3). Probe the real shape at runtime instead
of dying at a stale assumption.

Run these 2 cheap probes (≈ last 24h via the CLI timeframe flags) and bind the
results into variables the bootstrap interpolates:

**(a) App resolution probe — `fetch dt.entity.application` is PRIMARY (DV-3):**

<!-- VALIDATE: tenant-validated 2026-06-03 (EasyTrade → APPLICATION-5DCD0C470BC8680A) -->
```bash
DTCTL_TOKEN_STORAGE=file dtctl query -o json --plain \
  'fetch dt.entity.application | filter entity.name == "{APP_NAME}" or contains(entity.name, "{APP_NAME}") | fields id, entity.name | limit 5'
```

CANDIDATE LIST for resolution method, ordered `[dtctl-fetch (PRIMARY), MCP-fallback]`:
- If this returns ≥1 row → bind `APP_RESOLVER = "dtctl-fetch"`, pick the exact-name
  match (or single match) as `APP_ENTITY_ID`. **This is the working path on this
  tenant — do NOT call the MCP.**
- ONLY if it errors (not zero-rows) or the MCP is needed for disambiguation → fall
  back to `mcp__dynatrace__find_entity_by_name` (`APP_RESOLVER = "mcp-fallback"`).
  The MCP is unregistered on this tenant (X-2), so treat its absence as expected and
  proceed with the dtctl result.
- If BOTH yield nothing → emit `## Error: Application Not Found` (absence gated, not
  assumed).

**(b) Vitals metric probe — confirm the metric exists AND populates WITH `interval:` (DV-1):**

<!-- VALIDATE: tenant-validated 2026-06-03; interval:1h returns clean intervals, no-interval returns 0 -->
```bash
DTCTL_TOKEN_STORAGE=file dtctl query -o json --plain \
  --default-timeframe-start "{now-24h ISO}" \
  --default-timeframe-end   "{now ISO}" \
  'timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), interval:1h, by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

- If this returns rows with a non-empty `lcp` array → the vitals substrate is live;
  bind `VITALS_INTERVAL = "1h"` (the DV-1 fix). Proceed.
- If it returns rows only when grouped without the app filter (tenant-aggregate) →
  the metric is present but not app-dimensioned; set `APP_DIMENSIONED = false` and
  proceed to the documented tenant-aggregate degradation (see "Page / Geo
  Dimensions" below). This is a graceful no-op, never a failure.
- If it returns ZERO rows even unfiltered WITH `interval:1h` → emit
  `## Error: Application Not Reporting Vitals` (absence gated).

> **DV-1 — NON-NEGOTIABLE:** every windowed vitals `timeseries percentile(...)` in
> this skill (the probe above, Q1b, Q3, and all 6 W-vitals queries) MUST carry an
> explicit `interval:`. Proven live (2026-06-03): a windowed percentile timeseries
> with `interval:1h` returns ~24 clean intervals over a 24h window; the SAME query
> WITHOUT `interval:` returns **0 records**. Default `VITALS_INTERVAL = 1h`; derive
> a coarser/finer interval only if WINDOW_LEN ≫ 24h (target ~24 intervals/window).

**Probe summary (emit ONE line to the run log):**
`Probe: app→{dtctl-fetch|mcp-fallback} ({APP_ENTITY_ID}), vitals interval→{VITALS_INTERVAL}, app_dimensioned→{true|false}`

#### Q1 — Resolve APP entity ID from name (smartscape)

<!-- VALIDATE: tenant-validated 2026-05-28; per-tenant naming may vary -->
```dql
fetch dt.entity.application
| filter entity.name == "{APP_NAME}" or contains(entity.name, "{APP_NAME}")
| fields id, entity.name
| limit 5
```

**DV-3 — PRIMARY resolution is this `fetch dt.entity.application` query (dtctl),
NOT the MCP.** Phase 0b.0 already ran it; reuse that result. The MCP
`find_entity_by_name` is a *fallback only* and is unregistered on this tenant (X-2),
so do not call it when the dtctl fetch returns rows.

Pick the single best match. If 0 rows → exit with `## Error: Application Not Found`.
If multiple rows → log the candidates and pick the exact-match entry; if no exact
match, exit with `## Error: Ambiguous Application Name` and list candidates.
Store as `APP_ENTITY_ID`.

#### Q1b — Confirm metric data exists for the entity

**DV-1 — `interval:` REQUIRED.** Without it this windowed percentile timeseries
returns 0 records (proven live 2026-06-03). Use `{VITALS_INTERVAL}` (default `1h`,
bound in Phase 0b.0).

<!-- VALIDATE: tenant-validated 2026-06-03; interval:1h returns clean intervals over the window -->
```bash
DTCTL_TOKEN_STORAGE=file dtctl query \
  --default-timeframe-start "{now-7d ISO}" \
  --default-timeframe-end   "{now ISO}" \
  'timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), interval:{VITALS_INTERVAL}, by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

If zero records → exit with `## Error: Application Not Reporting Vitals` (the
entity exists but no LCP metric data in 7d). Extract `RECENT_LCP_INTERVALS`
(non-null array length) for the Appendix.

#### Q2 — IF `--vs-deploy`: resolve deployment event → derive windows

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch events, from: now() - 30d
| filter event.kind == "DEPLOYMENT_EVENT" or event.type == "CUSTOM_DEPLOYMENT"
| filter contains(toString(event.id), "{DEPLOY_ID}")
     or contains(toString(event.name), "{DEPLOY_ID}")
| fields timestamp, event.id, event.name, event.description
| sort timestamp desc
| limit 1
```

Extract `DEPLOY_TIMESTAMP`. Derive:

```
BASELINE.from = DEPLOY_TIMESTAMP minus 24h
BASELINE.to   = DEPLOY_TIMESTAMP
COMPARE.from  = DEPLOY_TIMESTAMP
COMPARE.to    = DEPLOY_TIMESTAMP plus 24h
```

If `now() - DEPLOY_TIMESTAMP < 1h`, **warn** in the report banner: the compare
window is shorter than 1 hour and the regression deltas may be noisy. Still proceed.

#### Q3 — Per-window sanity check (LCP metric intervals)

Run for BOTH windows in parallel. **DV-1 — `interval:` REQUIRED** (without it a
windowed percentile timeseries returns 0 records). Use `{VITALS_INTERVAL}`:

<!-- VALIDATE: tenant-validated 2026-06-03; interval:1h ⇒ ~24 intervals over a 24h window, 0 records WITHOUT interval -->
```bash
DTCTL_TOKEN_STORAGE=file dtctl query \
  --default-timeframe-start "{WINDOW.from}" \
  --default-timeframe-end   "{WINDOW.to}" \
  'timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), interval:{VITALS_INTERVAL}, by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

Extract `BASELINE_INTERVALS` and `COMPARE_INTERVALS` (non-null array length
per window). If either side has `intervals < 4`, set
`LOW_TRAFFIC_WARNING = true` and annotate the report.

Store everything as **DispatchContext JSON**:

```json
{
  "APP_NAME": "...",
  "APP_ENTITY_ID": "APPLICATION-XXXXXXXXXXXXXXXX",
  "APP_RESOLVER": "dtctl-fetch|mcp-fallback",
  "APP_DIMENSIONED": true,
  "APP_TYPE": "web|mobile|spa",
  "PAGE_FIELD": "page.url.path|view.name",
  "VITALS_INTERVAL": "1h",
  "WINDOW_LEN": "24h",
  "BASELINE": { "from": "ISO", "to": "ISO" },
  "COMPARE":  { "from": "ISO", "to": "ISO" },
  "WINDOW_SOURCE": "explicit|--vs-deploy",
  "DEPLOY_ID": "DEPLOY-XXXX or null",
  "DEPLOY_TIMESTAMP": "ISO or null",
  "BASELINE_INTERVALS": 0,
  "COMPARE_INTERVALS": 0,
  "LOW_TRAFFIC_WARNING": false,
  "SHORT_COMPARE_WARNING": false,
  "WITH_GEO": false,
  "CLEAN_MODE": false,
  "PDF_MODE": false,
  "FULL_APPENDIX": false
}
```

---

### Phase 0c: Reference Strategy

dt-dem-vitals is **reference-driven** — workers carry no hardcoded RUM DQL. Each
worker reads ONE targeted reference file at start (see Sub-Skills table), once per
subagent, derives the query patterns from it, and applies the dt-dem-vitals scoping
below. This keeps DQL current automatically.

**Orchestrator exceptions (inline only):** Phase 0b bootstrap (Q1–Q3) and Phase 1.5
Absence-Gate confirmation queries remain inline — they are orchestration primitives,
tiny and stable, and must run BEFORE workers dispatch.

Proceed from Phase 0b to Phase 1.

---

### Phase 1: Parallel Worker Dispatch

> **PRE-DISPATCH REALITY GATE (NON-NEGOTIABLE — added 2026-06-03):** Before dispatching ANY worker,
> confirm the Phase 0b bootstrap (and Phase 0b.0 probe) actually executed at least one SUCCESSFUL
> `dtctl query` that returned real data (rows, or a deliberate proven-zero) — NOT an `{ok:false}`
> error envelope, NOT an auth failure, NOT an unrun query. If the bootstrap could not execute a
> single successful query (dead auth, connection error, or every bootstrap query errored), **STOP
> HERE — do NOT fan out workers.** Emit the skill's auth/substrate-unreachable error banner (see
> ERROR HANDLING) and exit. Dispatching workers into a dead substrate is exactly what produces
> fabricated findings — a run that stops with a clear "could not reach the tenant" banner is
> correct; a run that fans out workers with no working auth invites confabulation. (The Phase 0b
> bootstrap queries are the proof — if they returned data, auth is live and dispatch is safe.)

**Dispatch ALL workers in ONE message with multiple Agent calls — true concurrency.**
Dispatching one-at-a-time defeats the architecture. The three workers run AT THE
SAME TIME.

Workers V (vitals), R (regressions-synthesizer), A (attribution) run concurrently
as subagents (`subagent_type: Explore`). The R worker depends on V's output, but
both V and A can fire in parallel; R is gated to start after V returns. In practice
the orchestrator dispatches V and A simultaneously, then dispatches R when V
completes (single-message R dispatch — do not also re-dispatch V or A).

**Speed model:** each worker reads ONE bundle of targeted reference files (5–20KB
each, once), derives its queries, and fires them in parallel. With the on-disk
dtctl token pre-warmed in Phase 0-auth, workers never block on auth. Expected
total: bootstrap + worker fan-out (parallel) + synthesis ≈ 5–7 min at medium effort.

**Worker-failure handling — NO silent serial fallback.** If a worker returns gaps
or fails, do NOT re-run its queries inline in the orchestrator thread — that
serializes the work AND pulls raw output into the orchestrator context. Instead:
**re-dispatch that ONE worker ONCE** (as a fresh Agent call, with a note on what
to fix), and if it still fails, record a `⚠️ gap` for that phase in the report.
The only queries the orchestrator runs itself are Phase 0b bootstrap and the
Phase 1.5 Absence-Gate confirmations.

#### Worker Prompt Template (applies to V, R, A)

```
You are a frontend regression analysis worker for dt-dem-vitals (Web Vitals
Regression Brief).

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed in "REFERENCE FILES" below. Read each file
ONCE. Do not read any other files. Do not re-read. Take the query patterns from
the reference; apply the DISPATCH CONTEXT scoping (APP_NAME, BASELINE, COMPARE,
PAGE_FIELD). This is the ONLY source of DQL truth — do NOT derive queries from
training knowledge or guess field names.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple `dtctl query` tool calls,
in parallel). Do not read files and query at the same time — read first, then
query.

RUM DATA MODEL — CRITICAL (tenant-validated 2026-05-28):
- VITALS are METRICS, queried via `timeseries` over `dt.frontend.web.page.*`.
  Group by `dt.entity.application` and filter by the resolved APP_ENTITY_ID.
  Inject windows via `--default-timeframe-start/--default-timeframe-end` CLI
  flags. NEVER mutate the DQL with `from:`/`to:`.
- **DV-1 — every vitals `timeseries percentile(...)` MUST include
  `interval:{VITALS_INTERVAL}`** (from the DISPATCH CONTEXT; default `1h`).
  Without it the windowed query returns 0 records. A zero-row result from a
  vitals query that LACKS `interval:` is a defect, not evidence of absence — add
  the interval and retry before concluding anything.
- DO NOT `fetch dt.rum.web.events` (does not exist) and DO NOT expect vital
  fields on `user.events` (the `user.events` data object exists but does NOT
  carry web_vitals.* fields).
- W-attribution uses `user.events` for per-page XHR + trace.id correlation
  (that path is valid and tenant-validated). It filters by
  `frontend.name == "{APP_NAME}"` and joins to spans via `trace.id`.
- Time field on user.events and spans is `start_time`. Logs use `timestamp`.
- Backend service name: getNodeName(dt.smartscape.service).

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

QUERIES TO RUN:
{QUERY_INSTRUCTIONS_FOR_THIS_WORKER}

RULES:
- EXPLORATION CAP: run ONLY the queries defined in "QUERIES TO RUN". No ad-hoc
  "let me also check…" queries. This cap is mandatory.
- PREFLIGHT: run `DTCTL_TOKEN_STORAGE=file dtctl query 'fetch user.events | limit 1'`.
  If it errors with an auth/connection problem (not DQL), return immediately with
  `<gap: dtctl-unavailable: {message}>` and STOP.
- Run all independent queries in parallel (multiple tool calls per message).
- Do NOT return raw rows. Internalize; summarize aggressively.
- Cap your PhaseResult at ~4KB. Top-10 of anything, not top-100.
- If a query fails with a SYNTAX/FIELD error, retry ONCE with corrected syntax
  (check: aliased bin not backticked; `start_time` for user.events/spans;
  single-quoted dtctl arg; `frontend.name` not `dt.entity.application`).
- AUTH IS THE ORCHESTRATOR'S JOB — workers NEVER run `dtctl auth login` OR
  `dtctl auth refresh`. If a dtctl call returns an auth error, verify ALL your
  dtctl calls have the `DTCTL_TOKEN_STORAGE=file` prefix. If they do and it
  still fails, return `<gap: auth>` and STOP — never attempt any auth recovery.
  An auth error is NEVER evidence of "no data".
- ERROR ≠ ABSENCE: a query that ERRORED tells you nothing about whether data
  exists. Only a SUCCESSFUL query returning zero rows is evidence of absence —
  and if that query was scoped (by page, geo, frontend.name…), CONFIRM with a
  broader unfiltered query before concluding "no events / not instrumented".
- PROOF OF ABSENCE (required): if you DO conclude "not present / no events",
  your PhaseResult MUST include the exact UNFILTERED confirmation query you
  ran and its zero-row result.
- Execute DQL via `DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'` — prefix EVERY
  dtctl call with `DTCTL_TOKEN_STORAGE=file` (no space between assignment and
  dtctl). **SINGLE QUOTES around the DQL**, always.
- NO BACKTICKS in DQL. For binned timeseries, ALIAS the bin
  (`by:{ ts = bin(start_time, 1h) } | sort ts asc`) — never `sort `bin(...)``.

RETURN only this PhaseResult shape (JSON). Nothing else.
{PHASE_RESULT_SCHEMA}
```

---

#### W-vitals (Vitals Worker)

**REFERENCE-DRIVEN — dt-dem-vitals carries NO RUM DQL beyond bootstrap.** Read
`~/.agents/skills/dt-obs-frontends/references/WebVitals.md` ONCE (all vital
metric names + percentile patterns). Read once per subagent; never re-read.

**dt-dem-vitals scoping/hygiene for EVERY vitals query (tenant-validated):**

- Vitals are **METRICS**, queried via `timeseries`. Do NOT use
  `fetch user.events` for vitals — that path was removed after 2026-05-28
  tenant validation (the vital fields don't exist as event fields).
- Filter by `dt.entity.application == "{APP_ENTITY_ID}"` (resolved in Phase 0b).
- Two windows per metric — same DQL run twice with different CLI timeframe
  flags. The DQL is window-agnostic; windows are injected via
  `--default-timeframe-start "{WINDOW.from}" --default-timeframe-end "{WINDOW.to}"`.
  Never mutate the DQL with `from:`/`to:`.
- **DV-1 — EVERY vitals timeseries MUST carry `interval:{VITALS_INTERVAL}`**
  (default `1h`, supplied in the DISPATCH CONTEXT). Without it the windowed
  percentile timeseries returns **0 records** over short windows — proven live.
  A vitals query without `interval:` is a defect; a zero-row result from such a
  query is NOT evidence of absence.
- Use `percentile(<metric>, 75)` for all vitals.
- Group by `dt.entity.application` only (v1). Geo split is `--with-geo`-gated v2.

**Queries to run (ALL in one parallel batch — 12 queries total: 6 vitals × 2 windows):**

For each of LCP, CLS, INP, FID, FCP, TTFB and each of {BASELINE, COMPARE},
issue the same `dtctl query` invocation with the appropriate timeframe flags.

Sample (LCP — substitute the metric name for each vital). **Note `interval:{VITALS_INTERVAL}` — DV-1, mandatory:**

<!-- VALIDATE: tenant-validated 2026-06-03; interval:1h ⇒ ~24 intervals/24h window, 0 records WITHOUT interval -->
```dql
timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75),
           interval:{VITALS_INTERVAL},
           by:{dt.entity.application}
| filter dt.entity.application == "{APP_ENTITY_ID}"
```

Invocation:

```bash
DTCTL_TOKEN_STORAGE=file dtctl query \
  --default-timeframe-start "{WINDOW.from}" \
  --default-timeframe-end   "{WINDOW.to}" \
  'timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75), interval:{VITALS_INTERVAL}, by:{dt.entity.application} | filter dt.entity.application == "{APP_ENTITY_ID}"'
```

Metric-name → alias map (for the 6 queries per window):

| Vital | Metric | Alias |
|---|---|---|
| LCP | `dt.frontend.web.page.largest_contentful_paint` | `lcp` |
| CLS | `dt.frontend.web.page.cumulative_layout_shift` | `cls` |
| INP | `dt.frontend.web.page.interaction_to_next_paint` | `inp` |
| FID | `dt.frontend.web.page.first_input_delay` | `fid` |
| FCP | `dt.frontend.web.page.first_contentful_paint` | `fcp` |
| TTFB | `dt.frontend.web.page.time_to_first_byte` | `ttfb` |

**Sparse-vital rule:** a `timeseries` that succeeds but returns zero records,
or returns records with an all-null array, counts as **omit-with-note** — NOT a
regression. Add `{vital, "no metric samples in window"}` to `omitted_vitals[]`.

**Geo split (only if `--with-geo` AND probe succeeds — v2):**

Probe (BASELINE window only, LCP only):

<!-- VALIDATE: dimension availability per-tenant; gate on probe -->
```dql
timeseries lcp = percentile(dt.frontend.web.page.largest_contentful_paint, 75),
           interval:{VITALS_INTERVAL},
           by:{dt.entity.application, geo.country.iso_code}
| filter dt.entity.application == "{APP_ENTITY_ID}"
| limit 1
```

If the probe errors with an unknown-dimension or returns zero rows → set
`geo_enabled = false`, add a gap note, skip geo for the rest of the run. If
the probe returns rows → run the same shape for both windows, capture top 10
countries by sample count.

**Regression delta computation rule (NEW):**

Each per-window `timeseries` returns ONE record per `dt.entity.application`
with the alias as an ARRAY of per-interval percentile values. To compute the
window's representative value:

1. Filter the array to non-null values.
2. Compute `percentile(75)` of the filtered array (preferred), or `mean()` if
   the array is too short (< 4 non-null intervals).
3. Delta = `compare_value - baseline_value`.
4. Apply the regression thresholds (see W-regressions).

This pooling is done in the W-vitals worker itself (it has the arrays in hand)
and the per-vital `baseline_value` / `compare_value` / `delta` is what is
passed to W-regressions.

**PhaseResult shape (v1 — APP-level):**

```json
{
  "app_entity_id": "APPLICATION-XXXXXXXXXXXXXXXX",
  "vitals_per_app": {
    "baseline": {
      "lcp_p75_ms": 2100, "cls_p75": 0.05, "inp_p75_ms": 180,
      "fid_p75_ms": 40,  "fcp_p75_ms": 1400, "ttfb_p75_ms": 600,
      "intervals_used": { "lcp": 24, "cls": 24, "inp": 22, "fid": 18, "fcp": 0, "ttfb": 0 }
    },
    "compare":  {
      "lcp_p75_ms": 2450, "cls_p75": 0.07, "inp_p75_ms": 260,
      "fid_p75_ms": 55,  "fcp_p75_ms": 1480, "ttfb_p75_ms": 640,
      "intervals_used": { "lcp": 24, "cls": 24, "inp": 23, "fid": 19, "fcp": 0, "ttfb": 0 }
    }
  },
  "omitted_vitals": [
    { "vital": "FCP", "reason": "no metric samples in either window" },
    { "vital": "TTFB", "reason": "no metric samples in either window" }
  ],
  "geo_enabled": false,
  "geo_split_lcp_baseline": [],
  "geo_split_lcp_compare":  [],
  "gaps": []
}
```

---

#### W-regressions (Regression Synthesizer)

**Reads NO reference file.** Consumes the V worker's output, applies the
dt-dem-vitals regression thresholds, and emits a regressed-vital list at the
APP level (v1). Runs no DQL.

**Regression thresholds — material delta gates:**

| Vital | Trigger |
|---|---|
| INP | `compare ≥ baseline × 1.10` AND `compare - baseline ≥ 50ms` |
| LCP | `compare - baseline ≥ 200ms` |
| CLS | `compare - baseline ≥ 0.05` |
| FID | `compare - baseline ≥ 25ms` |
| FCP | `compare - baseline ≥ 200ms` |
| TTFB | `compare - baseline ≥ 100ms` |

A vital is "regressed" if it crosses its threshold. The dominant vital is the
one with the largest normalized delta (delta / threshold).

**Volume gate:** if `BASELINE_INTERVALS < 4` OR `COMPARE_INTERVALS < 4` for a
given vital → exclude that vital with `"reason": "intervals<4"`.

**Sparse-vital handling:** vitals in `omitted_vitals[]` from W-vitals are
forwarded unchanged into the report (they are NOT regressions, but they ARE
documented in Appendix A so the reader knows they were attempted).

**Geo regression (v2, only when `geo_enabled == true`):** find the country
with the largest LCP delta crossing the LCP threshold; surface it as
`geo_regression`.

**PhaseResult shape (v1 — APP-level):**

```json
{
  "app_entity_id": "APPLICATION-XXXXXXXXXXXXXXXX",
  "regressed_vitals": [
    {
      "vital": "INP",
      "baseline_p75_ms": 180,
      "compare_p75_ms": 260,
      "delta_ms": 80,
      "delta_pct": 44,
      "triggered": true
    },
    {
      "vital": "LCP",
      "baseline_p75_ms": 2100,
      "compare_p75_ms": 2450,
      "delta_ms": 350,
      "delta_pct": 17,
      "triggered": true
    }
  ],
  "dominant_vital": "INP",
  "all_vitals_evaluated": [
    { "vital": "LCP", "triggered": true,  "delta_ms": 350 },
    { "vital": "CLS", "triggered": false, "delta": 0.02 },
    { "vital": "INP", "triggered": true,  "delta_ms": 80 },
    { "vital": "FID", "triggered": true,  "delta_ms": 15 },
    { "vital": "FCP", "triggered": false, "delta_ms": 80 },
    { "vital": "TTFB","triggered": false, "delta_ms": 40 }
  ],
  "excluded_vitals": [{ "vital": "FCP", "reason": "intervals<4" }],
  "omitted_vitals":  [{ "vital": "TTFB", "reason": "no metric samples in either window" }],
  "geo_regression": null,
  "gaps": []
}
```

---

#### W-attribution (Attribution Worker)

**REFERENCE-DRIVEN — read the bundle ONCE.** `RequestPerformance.md` (XHR
duration percentiles), `RequestTimingAnalysis.md` (TTFB decomposition into DNS /
connect / TLS / server / download), `TraceCorrelation.md` (frontend→backend join),
plus `entity-lookups.md` (service name resolution) from dt-obs-tracing. Read once
per subagent.

**Inputs:** the regressed-vital list from W-regressions (APP-level v1). Because
v1 has no per-page metric breakdown, W-attribution first identifies the top-N
candidate pages from `user.events` (largest XHR duration delta between windows,
or largest sample count on the most-trafficked URLs), then runs the
attribution queries below for those pages. **The worker MUST wait for
W-regressions to complete before firing its queries** — the orchestrator gates
the A worker dispatch on V+R completion.

**Candidate-page query (run FIRST, before the per-page queries below):**

<!-- VALIDATE: tenant-validated 2026-05-28 -->
```dql
fetch user.events, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
| filter frontend.name == "{APP_NAME}"
| filter characteristics.has_request == true
| summarize requests = count(),
            p75_duration_ms = percentile(duration, 75) / 1000000,
            by: {page = page.url.path}
| filter requests > 100
| sort p75_duration_ms desc
| limit 5
```

The top 5 pages by p75 request duration in COMPARE become the per-page
attribution targets.

**dt-dem-vitals scoping/hygiene for EVERY attribution query:**

- Frontend (user.events) queries: `frontend.name == "{APP_NAME}"` + page filter +
  COMPARE window.
- Backend (spans) queries: scope by `trace.id` (joined from the user.events side).
  NEVER guess a backend service ID and filter spans by it.
- Limit XHR top-N to 10 per page. Limit backend trace correlation to 50 rows per
  page (top by frontend duration).
- For TTFB-dominant regressions, run the timing-phase decomposition.

**Queries to run (per candidate page, ALL in one parallel batch):**

For each page returned by the candidate-page query above:

1. **Top XHRs by duration delta** (RequestPerformance.md + RequestTimingAnalysis.md):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch user.events, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
| filter frontend.name == "{APP_NAME}"
| filter characteristics.has_request == true
| filter performance.initiator_type == "xmlhttprequest" or performance.initiator_type == "fetch"
| filter page.url.path == "{REGRESSED_PAGE}"
| summarize p75_duration_ms = percentile(duration, 75) / 1000000,
            request_count = count(),
            by: {url.path}
| filter request_count > 20
| sort p75_duration_ms desc
| limit 10
```

2. **TTFB phase decomposition for the regressed page** (only when TTFB triggered):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch user.events, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
| filter frontend.name == "{APP_NAME}"
| filter characteristics.has_request == true
| filter page.url.path == "{REGRESSED_PAGE}"
| filter duration > 500000000
| fieldsAdd dns_ms = (performance.domain_lookup_end - performance.domain_lookup_start) / 1000000,
            connect_ms = (performance.connect_end - performance.connect_start) / 1000000,
            tls_ms = (performance.connect_end - performance.secure_connection_start) / 1000000,
            server_ms = (performance.response_start - performance.request_start) / 1000000,
            download_ms = (performance.response_end - performance.response_start) / 1000000
| summarize avg_dns = avg(dns_ms),
            avg_connect = avg(connect_ms),
            avg_tls = avg(tls_ms),
            avg_server = avg(server_ms),
            avg_download = avg(download_ms),
            requests = count(),
            by: {url.domain}
| sort avg_server desc
| limit 10
```

3. **Backend trace correlation** (TraceCorrelation.md "Backend Service Impact on
Frontend", adapted):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch user.events, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
| filter frontend.name == "{APP_NAME}"
| filter characteristics.has_request == true
| filter page.url.path == "{REGRESSED_PAGE}"
| filter isNotNull(trace.id)
| filter duration > 1000000000
| fields trace.id, frontend_duration = duration
| join [
    fetch spans, from: toTimestamp("{COMPARE.from}"), to: toTimestamp("{COMPARE.to}")
    | summarize backend_duration = sum(duration),
                services = collectDistinct(getNodeName(dt.smartscape.service)),
                by: {trace.id}
  ], on: trace.id, fields: {backend_duration, services}
| fieldsAdd backend_ratio_pct = 100.0 * backend_duration / frontend_duration
| summarize trace_count = count(),
            avg_backend_ms = avg(backend_duration) / 1000000,
            avg_backend_ratio = avg(backend_ratio_pct),
            top_services = collectDistinct(services)
| limit 1
```

**PhaseResult shape:**

```json
{
  "per_page_attribution": [
    {
      "page": "/checkout",
      "top_xhrs": [
        { "path": "/api/cart/totals", "p75_duration_ms": 820, "request_count": 4100 }
      ],
      "ttfb_decomposition": {
        "url.domain": "api.example.com",
        "avg_dns_ms": 2, "avg_connect_ms": 5, "avg_tls_ms": 8,
        "avg_server_ms": 380, "avg_download_ms": 12
      },
      "backend_correlation": {
        "trace_count": 1200,
        "avg_backend_ms": 540,
        "avg_backend_ratio_pct": 78,
        "top_services": ["cart-api", "pricing-svc"]
      }
    }
  ],
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator — run before trusting any "absent" finding)

Before using any worker result, scan every PhaseResult for absence claims ("not
present", "no events", "no traces", "0 …", "trace correlation disabled", etc.).
For EACH such claim:

1. Require the worker's **proof** (the unfiltered confirmation query + its
   zero-row result). If the worker did not attach it, the claim is invalid.
2. Run ONE cheap unfiltered confirmation query yourself. Examples:
   - vitals absence on a page: `fetch user.events, <window> | filter frontend.name == "{APP_NAME}" | summarize n=count(), by:{page.url.path} | filter page.url.path == "{PAGE}"`
     — if rows exist, events ARE present and the worker scoped wrong (likely missing
     `characteristics.has_page_summary` or vital `isNotNull` filter); re-dispatch.
   - trace correlation empty: `fetch user.events, <window> | filter frontend.name == "{APP_NAME}" | summarize total=count(), traced=countIf(isNotNull(trace.id))`
     — if `traced > 0` the join filter was wrong; re-dispatch W-attribution.
3. Only after a confirmation query genuinely returns zero across the broad scope
   may "absent" enter the report.

---

### Phase 2: Regression Classification & Synthesis (Orchestrator, Sequential)

After all workers return (and the Absence Gate has cleared any "absent" claims),
classify each regressed page into ONE of three classes:

#### Classification rules (apply in order — first match wins)

1. **`backend-bound`** — the regression's dominant vital is INP, LCP, or TTFB AND
   the attribution shows **either**:
   - `backend_correlation.avg_backend_ratio_pct >= 60` (backend dominates total
     duration), OR
   - `ttfb_decomposition.avg_server_ms` is at least 2× the sum of dns/connect/tls/download
     phases (server phase dominates TTFB).

2. **`network-bound`** — TTFB regressed AND the dominant phase in
   `ttfb_decomposition` is one of dns / connect / tls / download (not server). Or
   the geo split shows the LCP delta concentrated in 1–2 countries while
   home-country LCP is stable (CDN / regional egress issue).

3. **`frontend-rendered`** — default class when neither backend nor network
   classification triggers. Covers LCP-from-image-or-font, CLS regressions, INP
   regressions with low backend_ratio, FCP regressions without TTFB regression.

#### Synthesis output

For each regressed page produce:

```json
{
  "page": "/checkout",
  "dominant_vital": "INP",
  "deltas": { ... from W-regressions ... },
  "class": "backend-bound",
  "primary_cause_one_liner": "INP regression on /checkout driven by /api/cart/totals XHR p75 +320ms; backend service cart-api accounts for ~78% of frontend duration.",
  "evidence_refs": [
    { "phase": "V", "finding": "INP p75 180→260ms (+80ms, +44%)" },
    { "phase": "A", "finding": "/api/cart/totals p75 500→820ms (+320ms)" },
    { "phase": "A", "finding": "backend_ratio 78%; cart-api dominates trace" }
  ]
}
```

---

### Phase 3: Davis CoPilot Synthesis (Orchestrator, Sequential)

Pass the **classified regression list + attributions** to Davis CoPilot. Do NOT
pass raw worker findings.

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
WEB VITALS REGRESSION BRIEF: {APP_NAME} ({APP_ENTITY_ID})
WINDOWS: BASELINE {BASELINE.from}→{BASELINE.to} vs COMPARE {COMPARE.from}→{COMPARE.to}
WINDOW SOURCE: {explicit | --vs-deploy DEPLOY-XXXX}

APPLICATION-LEVEL REGRESSED VITALS:
- LCP: {b}→{c} (+{Δ} ms), triggered={true|false}
- CLS: {b}→{c} (+{Δ}), triggered={true|false}
- INP: {b}→{c} (+{Δ} ms), triggered={true|false}
- FID/FCP/TTFB: ...
Dominant vital: {vital}
Omitted vitals (sparse): {list}

CANDIDATE PAGES (per-page attribution, classified):

P1 [class={class}]: {page}
  Top XHR: {path} p75 +Xms
  TTFB decomposition: server={X}ms, dns={X}ms, tls={X}ms, download={X}ms
  Backend services in trace: [{svc1}, {svc2}]

P2 ...

QUESTIONS:
1. Given the dominant vital ({vital}) at the APP level, which candidate page
   most likely drives it? Cite the page-level evidence.
2. For each candidate page, who is the most likely remediation owner — frontend
   team, backend team owning the dominant service, or network/CDN team?
3. For the top page, suggest the single highest-leverage remediation step (do
   not list 5 options — name the one most likely to move the dominant vital).
4. Are there cross-page patterns suggesting a single root cause (e.g., shared
   third-party script regressed, CDN config change, common backend service)?
5. Confidence level in the attribution. What additional data would increase
   confidence?
```

Davis CoPilot response feeds the "Recommended Owners" and "Davis Synthesis"
sections.

---

### Phase 4: Report Generation

If CLEAN_MODE, build the sanitization map first (read dt-rca Phase 1.14 for the
canonical rules; RUM-specific targets below) and compose with sanitized values
from the start.

#### Phase 4.0 — RUM-specific sanitization targets (CLEAN_MODE only)

In addition to the dt-rca Phase 1.14 categories (services, hosts, clusters,
namespaces, owners, entity IDs, tenant URLs), dt-dem-vitals sanitizes:

| Category | Example Real Value | Example Sanitized Value |
|---|---|---|
| RUM application name | `www-checkout-prod` | `ap-shop-web` |
| Page paths | `/checkout/cart`, `/account/orders/12345` | `/section-a/page1`, `/section-b/item-XXX` |
| URL domains | `api.realcompany.com`, `cdn.realbrand.io` | `api.demo-shop.example`, `cdn.demo-shop.example` |
| URL query params | `?promo=BLACKFRIDAY` | (stripped — never relevant) |
| User-action names | `Add to Cart (Black Friday Promo)` | `Add to Cart` |
| Backend service names | `cart-api-prod` | `svc-cart` |
| Country codes (`geo.country.iso_code`) | (LEAVE AS-IS — too low-cardinality to be identifying) | (unchanged) |

**Page path normalization rule:** preserve structural shape (segments, presence of
IDs) so the report still reads as a coherent site map; only replace the names. A
two-segment path stays two segments; an ID-bearing path stays ID-bearing
(`/orders/{ID}`).

#### Phase 4.1 — Emit the report (canonical skeleton)

**Emit the report EXACTLY in the section order below.** Each fact has ONE canonical
home — never restate.

```markdown
# dt-dem-vitals — Web Vitals Regression Brief
## {APP_NAME}: {N} pages regressed | {DOMINANT_CLASS} dominant

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Baseline:** {BASELINE.from} → {BASELINE.to}  |  **Compare:** {COMPARE.from} → {COMPARE.to}
**Window Source:** {explicit | --vs-deploy DEPLOY-XXXX}  |  **Phases:** V·R·A

> _Warnings (if any):_ low traffic in {window}; compare window < 1h after deploy
> (deltas may be noisy).

---

## Executive Summary

### Critical Findings
[One short paragraph. Lead with regression count, dominant class, and the single
highest-impact page (largest normalized delta). State in ONE sentence — do not
re-explain later.]

### Impact Summary
| Metric | Baseline | Compare | Delta | Status |
|---|---|---|---|---|
| Pages with regression | 0 | {N} | +{N} | {triage flag} |
| Worst page (dominant vital) | {p75_b} | {p75_c} | +{Δ} | {good/needs/poor} |
| Sessions sampled (compare) | — | {N} | — | — |

### Business Impact
- {Affected pages}, {dominant vital}, {worst page delta}, {Google Search ranking
  exposure note if INP/LCP fell into "poor"}

### Immediate Action Required
| Priority | Action | Owner | Urgency |
|---|---|---|---|
[From Davis CoPilot for the top-N regressed pages — single highest-leverage step
each, plus owner team class.]

### Regression Class Distribution
```mermaid
pie title Regression Class Distribution
    "frontend-rendered" : {N}
    "backend-bound"     : {N}
    "network-bound"     : {N}
```

---

## Key Talking Points
| Topic | Detail |
|---|---|
[Exec one-liners — total regressed, dominant class, top-3 pages, primary owner.]

---

## Application-Level Vitals Delta (v1 — APP-level)

| Vital | Baseline (p75) | Compare (p75) | Delta | Triggered |
|---|---|---|---|---|
| LCP | 2,100 ms | 2,450 ms | +350 ms | ✅ |
| INP | 180 ms | 260 ms | +80 ms (+44%) | ✅ |
| CLS | 0.05 | 0.07 | +0.02 | — |
| FID | 40 ms | 55 ms | +15 ms | — |
| FCP | 1,400 ms | 1,480 ms | +80 ms | — |
| TTFB | 600 ms | 640 ms | +40 ms | — |

Vitals omitted (no metric samples): {FCP / TTFB / …}.

**Geo Breakdown** (only when `--with-geo` succeeded)

| Country | Baseline LCP p75 | Compare LCP p75 | Delta |
|---|---|---|---|

---

## Per-Page Attribution

For each candidate page (top-N by COMPARE-window p75 request duration):

### {N}. {page} — {class} | likely vital: {vital}

**Attributed Cause**

[Two-sentence explanation citing the page's top XHR, TTFB-server phase, or
backend service evidence. Class-specific.]

---

## Backend Attribution Table

ONE table across all candidate pages — never duplicate per page.

| Page | Top XHR | XHR p75 (ms) | TTFB Server (ms) | Backend Services in Trace | Backend Ratio % |
|---|---|---|---|---|---|

---

## Recommended Owners

Single canonical owner table — Davis CoPilot synthesis applied per page.

| Page | Class | Owner Team | Recommended Action |
|---|---|---|---|
| /checkout | backend-bound | Backend (cart-api) | Investigate /api/cart/totals p75 regression |
| /home | frontend-rendered | Frontend | Audit hero-image largest-paint candidate |
| /search | network-bound | Network/CDN | Investigate DNS / TLS regression for cdn.example |

---

## Davis Synthesis

[Davis CoPilot's cross-page pattern analysis, owner-assignment rationale, and
confidence note. If Davis unavailable, replace with a banner and a brief
evidence-only synthesis.]

---

## Appendix A: Investigation Details

| Field | Value |
|---|---|
| APP | {APP_NAME} ({APP_TYPE}) |
| Baseline events / pages | {N} / {N} |
| Compare events / pages | {N} / {N} |
| Window source | {explicit | --vs-deploy DEPLOY-XXXX} |
| Pages evaluated | {N} |
| Pages regressed | {N} |
| Pages excluded (low samples) | {N} |
| Sub-Skills Consulted | dt-obs-frontends (WebVitals, PageViewAnalysis, AdvancedPerformance, RequestPerformance, RequestTimingAnalysis, TraceCorrelation); dt-obs-tracing (entity-lookups) |
| Davis CoPilot | {Used / Unavailable} |

### dt-dem-vitals Telemetry
| Worker | Sub-Skill | Key Findings | Gaps |
|---|---|---|---|
| W-vitals | dt-obs-frontends/WebVitals | … | … |
| W-regressions | (synthesizer) | … | … |
| W-attribution | dt-obs-frontends/Trace+RequestTiming | … | … |

## Appendix B: Glossary
[Include ONLY if FULL_APPENDIX == true. Cover INP/LCP/CLS/FCP/TTFB definitions,
"p75", "frontend.name", "backend ratio", "TTFB server phase".]

## Appendix C: AI-Powered Analysis Capabilities
[Include ONLY if FULL_APPENDIX == true.]

## Appendix D: Query Exchange Log
[If FULL_APPENDIX == true: full DQL log with timing for all phases.
 If FULL_APPENDIX == false: one line → "Full query log: {LOG_PATH}".]

---

**Links:**
- [View RUM Application](https://{TENANT}.apps.dynatrace.com/ui/apps/dynatrace.rum/web/{APP_NAME})

---

*End of Report*
```

### Pre-Save Self-Check (run before Phase 6 writes anything)

If any item is "no", fix before writing:

- [ ] H1 title + `## {APP_NAME}: ...` H2 subtitle present
- [ ] Header: `Generated / Analyst / Environment / Baseline / Compare / Window
      Source / Phases`
- [ ] Executive Summary has all 5 sub-blocks (Critical Findings, Impact Summary
      table, Business Impact bullets, Immediate Action table, Regression Class
      Distribution pie)
- [ ] Key Talking Points present
- [ ] Application-Level Vitals Delta table present (one row per vital, all 6)
- [ ] Omitted vitals (sparse FCP/TTFB) listed under the table
- [ ] Per-Page Attribution has one subsection per candidate page (top-N by
      COMPARE p75 request duration)
- [ ] Geo Breakdown only when `--with-geo` probe succeeded
- [ ] Backend Attribution Table appears ONCE (never per-page)
- [ ] Recommended Owners table appears ONCE
- [ ] No DQL anywhere except Appendix D
- [ ] Appendix A present; B+C present only if `FULL_APPENDIX == true`; D present
      (full or log pointer)
- [ ] Footer is `**Links:**` + `*End of Report*`
- [ ] **MD-only by default** — only the `.md` is written unless `PDF_MODE == true`;
      any PDF check below is gated on `PDF_MODE`
- [ ] Filename will be `VITALS_{APP_SLUG}_{DATE}[_SANITIZED].md`
- [ ] **Mermaid lint** — pie labels are quoted strings, no `:` inside; graph node
      labels use `<br/>` not `\n`; no escaped `\"`. Sequence/gantt diagrams not
      used by default in this skill, but if added later follow dt-rcf
      label-sanitization rules.

---

### Phase 5: Reviewer Gate (Optional — only if `--review`)

dt-dem-vitals does NOT default to a reviewer subagent (the brief is a tight
deterministic diff, not a forensic narrative). If `--review` is passed, spawn ONE
reviewer subagent with this dt-dem-vitals-specific checklist:

```
1. Are all regressed pages attributed to exactly ONE class? (no missing, no double-class)
2. Did each backend-bound classification cite either backend_ratio >= 60 or
   server-phase dominance? (no class assigned without evidence)
3. Did each network-bound classification cite either a network-phase dominance
   or a geo concentration? (no class assigned without evidence)
4. Is the Recommended Owners table populated for every regressed page?
5. Are the windows non-overlapping and consistent across all sections of the report?
6. Did the report cite Davis CoPilot's confidence level (or note unavailability)?
```

Verdict ship / revise / reject follow the dt-rca-swarm rules.

---

### Phase 6: Save Report (Markdown)

Run the Pre-Save Self-Check (end of Phase 4) before writing anything.

**Markdown is the canonical deliverable. Default: write the `.md` ONLY.** The
PDF is opt-in via `--pdf` (or legacy `pdf=true`) and must NEVER block or fail a
run.

**Default (PDF_MODE == false):**
- Write `VITALS_{APP_SLUG}_{DATE}.md` (or `..._SANITIZED.md` in CLEAN_MODE). Done.

**Only if PDF_MODE == true:** attempt the inherited dt-rca PDF path — **identical
to dt-rca Phases 3+4** (read `~/.claude/skills/dt-rca/SKILL.md` for filename
generation, the `md-to-pdf` command, the intermediate `_pdf.md` strategy
(Mermaid → ASCII conversion), and cleanup). NEVER use the dt-rca
`Problem_{ID}_Analysis_Report` filename — dt-dem-vitals reports are `VITALS_*`.
**Wrap the PDF step so a missing `md-to-pdf` engine logs ONE line
(`PDF engine unavailable — Markdown only`) and continues without error.** A
failed or unavailable PDF engine never aborts the run; the `.md` is already
written and is the deliverable.

Filenames:

```
Default (MD-only): VITALS_{APP_SLUG}_{DATE}.md
With --pdf:        VITALS_{APP_SLUG}_{DATE}.md  +  VITALS_{APP_SLUG}_{DATE}.pdf
Clean:             VITALS_{APP_SLUG}_{DATE}_SANITIZED.md  (+ ...SANITIZED.pdf if --pdf)

APP_SLUG examples:
  APP:"www-checkout"          → www_checkout
  APP:"www-checkout" --vs-deploy DEPLOY-83A21F → www_checkout_vsDEPLOY83A21F
  APP:"mobile-ios"            → mobile_ios
```

**ASCII conversion targets (only relevant when `--pdf`, for `_pdf.md`):** the pie
chart in Executive Summary becomes a horizontal bar block:

```
Regression Class Distribution
  frontend-rendered  [######################]   12
  backend-bound      [#######............... ]    4
  network-bound      [##....................]    1
```

---

### Phase 7: Output Summary

```markdown
## dt-dem-vitals Brief Generated

| Format | Filename |
|---|---|
| Markdown | VITALS_{APP_SLUG}_{DATE}.md |
| PDF | VITALS_{APP_SLUG}_{DATE}.pdf (or "skipped (MD-only default; pass --pdf to enable)") |
| Log | VITALS_{APP_SLUG}_{DATE}.log |

**APP:** {APP_NAME}
**Baseline → Compare:** {BASELINE.from..BASELINE.to} → {COMPARE.from..COMPARE.to}
**Window source:** {explicit | --vs-deploy DEPLOY-XXXX}
**Pages regressed:** {N}  |  **Dominant class:** {class}

### Top 3 Regressions
| Rank | Page | Class | Dominant Vital | Delta |
| 1 | … | … | … | … |

### Davis Synthesis
- **Cross-page pattern:** {one_line or "none identified"}
- **Top remediation:** {one_line}

### dt-dem-vitals Telemetry
| Worker | Sub-Skill | Findings | Gaps |
| W-vitals | dt-obs-frontends/WebVitals | … | … |
| W-regressions | (synthesizer) | … | … |
| W-attribution | dt-obs-frontends/Trace+RequestTiming | … | … |
```

If CLEAN_MODE, append the sanitization key to console (never to file) per dt-rca
rules.

---

## CONSTRAINTS / KNOWN LIMITS (v1)

1. **v1 uses `dt.frontend.web.page.*` metrics, not RUM events.** Vitals are
   surfaced via `timeseries percentile(<metric>, 75) by:{dt.entity.application}`
   with windows injected via CLI timeframe flags. The earlier
   `fetch user.events | filter isNotNull(web_vitals.*)` path was removed after
   2026-05-28 tenant validation showed those fields do not exist on
   `user.events`, and `dt.rum.web.events` is `UNKNOWN_DATA_OBJECT`.

2. **APP-level granularity only (v1).** The metric family on this tenant carries
   `dt.entity.application` as the primary dimension. Per-page vitals breakdown
   from the metric is not in v1 — W-attribution provides per-page context via
   `user.events` instead.

3. **Geo split is v2 (`--with-geo` flag, gated on a probe).** If
   `dt.frontend.web.page.*` does not expose `geo.country.iso_code` as a
   dimension on the tenant, the flag is rejected and the run proceeds
   APP-level.

4. **FCP and TTFB may be sparse.** Both metrics exist on the tenant but
   returned empty arrays in the 1h validation window. The skill must handle
   empty `timeseries` results gracefully (omit the vital, document in
   Appendix A, NEVER fail the run).

5. **W-attribution still uses `user.events`.** That path was tenant-validated
   and is the only way to get per-page XHR + trace.id correlation. Do not
   replace it with metric queries.

6. **APP_NAME → APP_ENTITY_ID resolution is required.** Every vitals query
   filters by `dt.entity.application == "{APP_ENTITY_ID}"`, never by name.
   Phase 0b Q1 resolves the entity; if ambiguous, the run aborts with a
   candidate list.

---

## QUALITY RULES (NON-NEGOTIABLE)

### Inherited from dt-rca / dt-rcf

**Rules from dt-rca's "RULES (NON-NEGOTIABLE)" apply unchanged.** They cover:
no skipped data collection, Mermaid-in-MD / ASCII-in-PDF, executive audience,
Appendix-D query logging, compact diagrams, two-file PDF strategy, clean-mode
zero-leaks / consistency / plausibility, and the DQL backtick / timestamp /
count-alias, canonical-link, and frontend-name (RUM) rules.

### New in dt-dem-vitals — regression engine

1. **APP_NAME resolves to `APP_ENTITY_ID` (a Smartscape application entity ID)
   in Phase 0b.** Vitals (metric) queries filter by
   `dt.entity.application == "{APP_ENTITY_ID}"`. W-attribution queries against
   `user.events` filter by `frontend.name == "{APP_NAME}"`. NEVER mix the two:
   metrics use the entity ID, events use the name.

2. **Two-window discipline.** Every metric query runs TWICE (once per window)
   in parallel. Windows are injected via `--default-timeframe-start` /
   `--default-timeframe-end` CLI flags, NOT by mutating the DQL. Never attempt
   a single-query "join two windows" — silent percentile misalignment risk.

3. **Material-delta thresholds are gates, not suggestions.** A page is regressed
   only if it crosses the threshold for at least one vital. Do not surface
   "regressions" that fall below threshold; do not invent new thresholds.

4. **Per-vital minimum sample size of 50 per window.** Pages below the threshold
   are excluded with `"reason": "samples<50"` and called out in Appendix A. A
   regression brief built on noise is worse than no brief.

5. **Classification is single-class.** Every regressed page gets exactly one of
   `frontend-rendered`, `backend-bound`, `network-bound`. The classification
   rules are ordered (backend > network > frontend); the first match wins. Never
   tag a page with two classes.

6. **Backend attribution is trace-driven only.** Never claim a backend service
   caused a regression without a trace-id join showing that service in the trace
   set. Suspicion is not evidence.

7. **Davis CoPilot receives classified regressions, never raw vitals tables.**
   Phase 3 input is the classified list + attributions; passing raw worker
   findings degrades the synthesis quality.

8. **One canonical home per fact.** Vitals deltas → Per-Page Detail only.
   Backend attribution → Backend Attribution Table only (the page-detail
   subsection cites it). Owner assignments → Recommended Owners only. Never
   restate the same data across sections.

9. **Filename = `VITALS_{APP_SLUG}_{DATE}[_SANITIZED]`** — NEVER the dt-rca
   `Problem_{ID}_Analysis_Report` or dt-rcf `RCF_*` name. The slug is the app
   name, lowercased, non-alphanumerics replaced with `_`.

10. **Parallel worker dispatch in single message** — V and A workers MUST be
    dispatched in ONE message; R worker dispatches in a second message after V
    completes. Sequential dispatch (V → wait → A → wait → R) defeats the
    architecture.

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section "PDF
DIAGRAM RULES". The pie chart used in this skill's Executive Summary becomes a
horizontal-bar ASCII block in the PDF (sample above in Phase 6).

---

## MERMAID SYNTAX RULES

Inherits dt-rca's "MERMAID SYNTAX RULES" (`A["Text"]` not `A[\"Text\"]`; edge
labels `-->|"Text"|`; no escaped quotes). PLUS the dt-dem-vitals pie-chart
rule: pie labels MUST be quoted strings with no internal `:` (the `:` is the
label→value delimiter). Sanitize labels before emitting (`backend-bound (cart-api)`
→ `backend-bound`).

---

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section "DIAGRAM
SIZING RULES". Pie chart has at most 3 slices (the three classes); no class with
zero count gets a slice (omit it from the pie).

---

## ERROR HANDLING

### Application Not Found

```markdown
## Error: Application Not Found

RUM application `{APP_NAME}` did not resolve via `fetch dt.entity.application`
(PRIMARY) and returned 0 events in the last 7 days.

**Possible causes:**
- Name spelled differently in Dynatrace (use exact `entity.name` / `frontend.name` from the UI)
- Application was renamed or deprecated
- Wrong tenant / context (check `dtctl auth status`)

**Try:** re-run with a partial name — Phase 0b.0 already does a
`contains(entity.name, …)` match via `fetch dt.entity.application`. Only if the
Dynatrace MCP is registered does `mcp__dynatrace__find_entity_by_name` provide a
fallback; on this tenant the MCP is unregistered (X-2), so the dtctl fetch is the
authoritative resolution path.
```

### --vs-deploy not found

```markdown
## Error: Deployment Event Not Found

Deployment ID `{DEPLOY_ID}` was not found in the last 30 days of events.

**Possible causes:**
- ID typo or wrong tenant
- Deployment outside 30-day retention
- The integration that emits CUSTOM_DEPLOYMENT events isn't wired for this app

**Fix:** Re-run with explicit `BASELINE:"..."` and `COMPARE:"..."` windows.
```

### Worker fails entirely

Orchestrator marks the corresponding section with ⚠️, notes the gap in Appendix
A, and continues. Does not auto-retry — a second worker would likely fail for
the same reason.

### Davis CoPilot unavailable

Generate the report from the classified regression list alone. Add a banner to
"Davis Synthesis":

```
> **Note:** Davis CoPilot synthesis unavailable. Owner assignments are derived
> from class-based defaults (backend-bound → owning backend team; frontend-rendered
> → frontend team; network-bound → network/CDN team) without cross-page pattern
> analysis.
```

### No regressions found

The brief is still generated, with the executive summary stating "no material
regressions detected across {N} pages evaluated", a single Per-Page Detail
showing the top 5 pages with smallest deltas (proof the diff ran), and a
shortened Recommended Owners table (omitted).

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously until the
report is saved.

**Do not stop for confirmation between phases. Do not ask questions. Generate
the complete brief.**
