---
name: dt-deploy-risk
description: >
  Deployment Risk Scorecard for Dynatrace. Given a DEPLOY:<event-id> anchor (or
  SERVICE:"<name>" VERSION:"<v>" fallback), measures the delta between a
  configurable pre-deploy baseline window and the equivalent post-deploy window
  across RED metrics, exception types, log patterns, and downstream dependency
  health. Composes dt-obs-services, dt-obs-tracing, dt-obs-problems, and
  dt-obs-logs in a parallel worker dispatch (one message, four agents), scores
  the deploy 0-100 via a weighted rubric (error 0.35 / latency 0.25 / new
  exceptions 0.25 / dep churn 0.15), assigns a GO / HOLD / ROLLBACK verdict,
  passes the scorecard to Davis CoPilot for recommendation synthesis, and
  emits a Markdown + PDF report.
---

# dt-deploy-risk — Dynatrace Deployment Risk Scorecard

Post-deployment delta analyzer. Compares a configurable BASELINE window
(default 24h before T0) against the matching POST window (T0 → T0 + baseline,
or `now()` if the deploy is too recent), scores the deploy on a weighted
rubric, and emits a defensible GO / HOLD / ROLLBACK verdict with evidence.

## Usage

```
/dt-deploy-risk DEPLOY:abc123                                       # Deploy event ID anchor
/dt-deploy-risk SERVICE:"checkout-api" VERSION:"v2.41.0"            # Service + version fallback
/dt-deploy-risk DEPLOY:abc123 --baseline 7d                         # 7-day baseline window
/dt-deploy-risk SERVICE:"payments" VERSION:"build-7714" --baseline 4h
/dt-deploy-risk DEPLOY:abc123 -clean                                # Sanitized report
```

## Arguments

| Arg / flag | Default | Description |
|---|---|---|
| `DEPLOY:<event-id>` | preferred | Resolves the deploy event directly; unambiguous |
| `SERVICE:"<name>" VERSION:"<v>"` | fallback | Derives deploy event by version-tag match in last 48h |
| `--baseline <duration>` | `24h` | Window length each side of T0. Accepts `4h`, `24h`, `3d`, `7d` |
| `-clean` | off | Sanitize per dt-rca Phase 1.14 |
| `appendix=true` | off | Full Appendix query log |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with `CUSTOM_DEPLOYMENT` or `DEPLOYMENT_EVENT` ingestion
- dynatrace-for-ai skills: `dt-obs-services`, `dt-obs-tracing`, `dt-obs-logs`, `dt-obs-problems`
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-red        | `dt-obs-services/references/service-metrics.md` + runtime ref |
| W-exceptions | `dt-obs-tracing/references/failure-detection.md` |
| W-logs       | `dt-obs-logs/SKILL.md` |
| W-deps       | `dt-obs-tracing/references/failure-detection.md` (outbound client-span patterns) |

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0a: Anchor Parser

Parse `DEPLOY:<id>` OR `SERVICE:"<name>" VERSION:"<v>"` (not both). Extract `--baseline` duration.

**Fail-fast:** if no anchor parseable, or both forms supplied → print usage and exit.

### Phase 0b: Context Bootstrap (Sequential)

**If ANCHOR_TYPE == DEPLOY_EVENT:**
```dql
fetch events, from:now()-30d
| filter (event.kind == "DEPLOYMENT_EVENT" or event.type == "CUSTOM_DEPLOYMENT")
| filter event.id == "{ANCHOR_VALUE}"
| sort timestamp desc | limit 1
```
Extract: `T0` (timestamp), `ROOT_SERVICE_ID`, `VERSION`, `DEPLOY_NAME`.

**If ANCHOR_TYPE == SERVICE_VERSION:**
1. Resolve service entity via `mcp__dynatrace__find_entity_by_name`
2. Find deploy event by version-tag match
3. **Fail-fast:** if 0 rows OR >1 distinct `event.id` within 48h → abort with error

**Compute windows:**
```
BASELINE_WINDOW = { from: T0 - BASELINE_DURATION, to: T0 }
POST_WINDOW     = { from: T0, to: min(T0 + BASELINE_DURATION, now()) }
POST_COVERAGE_PCT = floor((now() - T0) / BASELINE_DURATION * 100)
```

If `POST_COVERAGE_PCT < 25` → abort with "Insufficient Post-Deploy Window" error.

**Resolve runtime + PGI (ONE combined span query):**
```dql
fetch spans, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
| filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
| summarize spancount = count(),
            by: { dt.entity.process_group_instance, k8s.namespace.name, k8s.workload.name }
| sort spancount desc | limit 1
```

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

Workers W-red, W-exceptions, W-logs, W-deps run concurrently. Each worker executes queries TWICE (BASELINE + POST) in ONE parallel batch, computes deltas internally, returns only deltas.

**W-red — RED metrics delta (span-derived, NOT dt.service.request.* metrics):**
```dql
fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
| filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
| filter request.is_root_span == true
| summarize reqs=count(), fails=countIf(request.is_failed==true),
            p95_ms=percentile(duration,95)/1e6, p99_ms=percentile(duration,99)/1e6
| fieldsAdd err_pct=(fails*100.0)/reqs
```

**W-exceptions — new exception types (set-difference POST − BASELINE):**
- Build BASELINE exception-type set and POST exception-type set separately
- Return `NEW_TYPES = POST_TYPES - BASELINE_TYPES` only
- Pre-existing exceptions are NOT regressions

**W-logs — new error/warn log patterns (set-difference):**
- Scope with 3-way OR: Smartscape + `dt.source_entity` + k8s namespace+container fallback
- Pattern key = `substring(content, 0, 80)` (fold near-duplicates)
- Return `NEW_PATTERNS = POST_PATTERNS - BASELINE_PATTERNS` only

**W-deps — downstream dependency health delta:**
- For each downstream: NEW (appeared in POST), REMOVED (dropped from POST), latency regression (>25%), error regression (+1pp)
- Return only changed dependencies

### Phase 1.5: Absence Gate

For any "no new exceptions / no new logs / no traffic" claim: require worker proof + run one unfiltered confirmation. A worker's empty result is never a conclusion.

### Phase 2: Scorecard Computation

**Penalty rubric:**

| Component | Weight | Penalty logic |
|---|---|---|
| error_delta_penalty | 35% | d = err_pct_post - err_pct_baseline; 0pp→p=0, ≤0.5pp→p=20, ≤2pp→p=50, ≤5pp→p=75, >5pp→p=100 |
| latency_delta_penalty | 25% | use max(p95_delta_pct, p99_delta_pct); ≤10%→p=20, ≤25%→p=50, ≤50%→p=75, >50%→p=100 |
| new_exception_penalty | 25% | n=0→p=0, n=1 small→p=30, n≤2→p=60, n≤5→p=85, n>5→p=100 |
| dep_churn_penalty | 15% | churn=0→p=0, 1→p=30, ≤3→p=60, ≤5→p=85, >5→p=100 |

**Gap rule:** worker returned `<gap>` → set penalty = 50 (flagged in report).

**Score formula:**
```
score = 100 - (error_delta_penalty * 0.35)
            - (latency_delta_penalty * 0.25)
            - (new_exception_penalty * 0.25)
            - (dep_churn_penalty * 0.15)
# round to nearest integer; clamp [0, 100]
```

**Verdict:**
```
score ≥ 80  → GO        (continue rollout)
50 ≤ score < 80 → HOLD  (manual review required)
score < 50  → ROLLBACK  (revert recommended)
```

**POST_IS_PARTIAL adjustment:** if `POST_COVERAGE_PCT < 50`, downgrade verdict one tier (GO→HOLD, but NEVER auto-escalate HOLD→ROLLBACK from partial data).

### Phase 3: Davis CoPilot Synthesis

Pass scorecard to Davis CoPilot (NOT raw PhaseResults). Ask for: verdict validation, failure-mode classification, highest-leverage next steps.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 title + H2 subtitle with VERDICT
2. Header: Generated / Analyst / Environment / Deploy Event / T0 / Baseline / Windows / Post coverage
3. Executive Verdict (one-sentence callout + Recommended Action table)
4. Score Breakdown table (all four components + weights + Total row)
5. Before / After Metrics (RED summary + post-deploy burn chart + runtime metrics)
6. New Exceptions (post only — set-difference)
7. New Log Patterns (post only — set-difference)
8. Dependency Impact (latency / error / churn subsections)
9. Davis CoPilot Synthesis
10. Appendix A: Investigation details + worker telemetry
11. Links + *End of Report*

**Filenames:**
```
Normal: DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}.md / .pdf
Clean:  DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **Deploy event resolution is required** — no scorecard without a confirmed `CUSTOM_DEPLOYMENT` / `DEPLOYMENT_EVENT`.
2. **Symmetric windows** — BASELINE_WINDOW and POST_WINDOW must be the same duration (POST may be truncated at `now()`).
3. **Delta-only worker outputs** — workers return delta-shaped PhaseResults, never raw BASELINE + POST rows.
4. **Set-difference is the unit** — exception types and log patterns scored on POST − BASELINE set, NOT absolute counts.
5. **Parallel dispatch in single message** — all 4 workers in ONE orchestrator message.
6. **Penalty cap** — no penalty exceeds 100. Rubric is the ONLY scoring path.
7. **Partial-window downgrade, never auto-rollback** — GO→HOLD at <50% coverage, but never HOLD→ROLLBACK.
8. **Gap penalty = 50** — never treat gap as clean (0) or rollback-bias (100).
9. **Filename = `DEPLOY_RISK_*`** — never dt-rca or dt-rcf naming.

## Real-World Use Cases

1. **CI pre-merge gate** — webhook runs skill 30 min post-deploy; non-GO score blocks next promotion.
2. **Manual post-deploy sanity** — engineer runs it after their own deploy as self-check before EOD.
3. **Rollback evidence** — scorecard is the audit trail attached to incident retro.
4. **Multi-service blast** — large feature rollout touched 5 services; one invocation per deploy event.
5. **Customer success story** — show customer how to wire into CD pipeline as Site Reliability Guardian companion.

## Substrate Notes (Validated 2026-05-28)

- `CUSTOM_DEPLOYMENT` events confirmed available (1933 in last 3d, ArgoCD origin on validation tenant)
- 48h ambiguity fail-fast on SERVICE+VERSION fallback (don't auto-pick when multiple deploys match)
- Span-derived RED required (NOT `dt.service.request.*` — `dt.entity.service` doesn't exist for OTel)
