---
name: dt-slo-burn
description: >
  Error-Budget Burn Briefing for Dynatrace SLOs. Computes multi-window
  burn-rates per the Google SRE Workbook (fast + slow burn), identifies top
  contributing services / endpoints / users, surfaces correlated open Davis
  problems, and (optionally) forecasts budget-exhaustion datetime via
  predictive analytics. Produces a polished MD + PDF briefing with verdict
  ladder (GO / SLOW BURN / FAST BURN / FREEZE). Use when an SLO is degraded,
  a release is gated on error-budget posture, or a stakeholder needs a sourced
  burn briefing.
---

# dt-slo-burn — Dynatrace Error-Budget Burn Briefing

Agentic burn-rate analysis skill. Resolves an SLO, dispatches parallel workers
to compute multi-window burn-rates, top contributors, and correlated Davis
problems (optionally a budget-exhaustion forecast), synthesizes a verdict via
Davis CoPilot, and produces a complete briefing (Markdown + PDF).

## Usage

```
/dt-slo-burn SLO:"checkout-availability"                        # 1h horizon (default)
/dt-slo-burn SLO:"checkout-availability" HORIZON:6h             # 6h horizon
/dt-slo-burn SLO:"a3f2c8e1-..." HORIZON:24h --forecast          # 24h + exhaustion forecast
/dt-slo-burn SLO:"login-latency-99p" HORIZON:30d --forecast     # 30d compliance + forecast
/dt-slo-burn SLO:"checkout-availability" -clean                 # Sanitized briefing
```

## Arguments

| Arg / flag | Default | Description |
|---|---|---|
| `SLO:"<id-or-name>"` | required | Opaque SLO UUID OR human display name. Names resolved via `dtctl get slo` + exact-then-fuzzy match. |
| `HORIZON:1h\|6h\|24h\|30d` | `1h` | Short-window horizon for burn evaluation |
| `--forecast` | off | Add W-forecast worker (budget-exhaustion datetime + confidence band) |
| `-clean` | off | Sanitize all identifying names per dt-rca Phase 1.14 |
| `appendix=true` | off | Full Appendix A query log |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with at least one **Custom SLI** SLO (template-based SLOs not supported in v1)
- dynatrace-for-ai skills: `dt-obs-services`, `dt-obs-problems`, `dt-obs-predictive-analytics` (for `--forecast`)
- `md-to-pdf` for PDF output

## Multi-Window Burn-Rate Table (Google SRE Workbook ch.5)

| BURN_HORIZON | Long window | Short window | Threshold (×) | Budget burned if fired |
|---|---|---|---|---|
| 1h  | 1h  | 5m  | 14.4 | ~5% in 1h  |
| 6h  | 6h  | 30m | 6    | ~10% in 6h |
| 24h | 24h | 2h  | 3    | ~10% in 24h |
| 30d | 30d | 6h  | 1    | budget tracking only |

Alert fires only when BOTH long AND short windows cross threshold.

## Sub-Skills Loaded Per Phase

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-burn         | `dt-obs-services/references/service-metrics.md` |
| W-contributors | `dt-obs-services/references/service-metrics.md` |
| W-davis        | `dt-obs-problems/references/problem-correlation.md` |
| W-forecast     | `dt-obs-predictive-analytics/references/forecasting-analyzer.md` (only if `--forecast`) |

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh` (silent, on-disk token). Workers never authenticate.

### Phase 0a: Argument Parser

Parse `SLO:`, `HORIZON:`, `--forecast`, `-clean`. Detect SLO anchor kind (UUID vs name).

### Phase 0b: SLO Bootstrap (Sequential)

**SLOs are settings resources — NOT Grail data objects. Never `fetch dt.slo`.**

Step 1: Resolve SLO_ID
```bash
dtctl get slo --plain   # list; parse with parsed.get("result", parsed)
```
Exact-match on `name` field first; fuzzy substring if no exact match. >1 match → list candidates and exit.

Step 2: Fetch SLO detail
```bash
dtctl describe slo {SLO_ID} -o json --plain
```
Extract: `SLO_NAME`, `SLO_TARGET_PCT`, `SLO_WARNING_PCT`, `SLO_EVAL_WINDOW`, `SLI_INDICATOR_DQL` (from `customSli.indicator` — note lowercase `l`).

**HARD FAIL if `SLI_INDICATOR_DQL` is null** — template-based SLO, not supported.

Step 3: Derive timeframe ISO strings for LONG and SHORT windows.

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

**W-burn — Multi-window Burn-Rate:**

Execute the SLO's `SLI_INDICATOR_DQL` verbatim over each burn window:
```bash
dtctl query '<SLI_INDICATOR_DQL verbatim>' \
  --default-timeframe-start '<ISO start>' \
  --default-timeframe-end   '<ISO end>' \
  -o json --plain
```

**NEVER mutate the indicator DQL.** Vary timeframe via CLI flags only.

Burn-rate formula:
```
budget         = 100 - SLO_TARGET_PCT
window_sli     = mean(non_null(flatten(sli arrays)))
error_rate_pct = 100 - window_sli
burn_rate      = error_rate_pct / budget
```

Verdict logic (requires BOTH long AND short to fire):
- Both ≥ threshold → `FIRING`
- Long ≥, short < → `RECOVERING`
- Long <, short ≥ → `SPIKE_ONLY`
- Both < → `OK`
- Either has null sli pool → `NO_TRAFFIC`

**W-contributors:** Top endpoints, callees, consumers from span data.

**W-davis:** Active Davis problems on SLO-affected entities.

**W-forecast (if --forecast):** Read `forecasting-analyzer.md`. Build training series by executing SLI indicator over forecast training window. Call `timeseries-forecast` tool. Project budget-exhaustion datetime.

### Phase 1.5: Absence Gate

For any "no failures / no problems / no traffic" claim: require worker proof + run one unfiltered confirmation query.

**Special case:** `alert_state == "NO_TRAFFIC"` → verdict is `NO TRAFFIC` (not OK). Skip burn-rate table.

### Phase 2: Synthesis & Verdict Ladder

```
alert_state == "NO_TRAFFIC"           → NO TRAFFIC (SLI denominator zero)
alert_state == "FIRING" + horizon 1h/6h → FAST BURN / FREEZE
alert_state == "FIRING" + horizon 24h/30d → SLOW BURN
alert_state == "RECOVERING"           → SLOW BURN (recovering)
alert_state == "SPIKE_ONLY"           → SPIKE — MONITORING
alert_state == "OK"                   → GO
```

Forecast overlay: if projected exhaustion is within HORIZON → escalate verdict one step.

### Phase 3: Davis CoPilot Synthesis

Pass verdict + top contributors + forecast to Davis CoPilot. Ask for: likely root cause, specific remediation for top failing endpoint, freeze vs monitor recommendation.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 title + H2 subtitle with VERDICT
2. Header: Generated / Analyst / Environment / SLO Anchor / Target / Eval Window / Burn Horizon
3. Executive Verdict (one sentence + optional forecast banner)
4. Burn-Rate Table (long + short window per pair; skip if NO TRAFFIC)
5. Top Contributors (Endpoint / Dependency / Consumer subsections)
6. Forecast (only if --forecast; otherwise "not requested" line)
7. Davis Synthesis: Correlated Active Problems + Davis CoPilot Assessment
8. Recommended Actions (table + Decision guidance bullets)
9. Appendix A: SLI expression verbatim + worker telemetry + query log
10. Links + *End of Briefing*

**Filenames:**
```
Normal: SLO_BURN_{SLO_SLUG}_{DATE}.md / .pdf
Clean:  SLO_BURN_{SLO_SLUG}_{DATE}_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **`fetch dt.slo` does NOT exist** — use `dtctl get slo` + `dtctl describe slo`.
2. **Never mutate the indicator DQL** — vary evaluation window via `--default-timeframe-*` flags only.
3. **Multi-window burn-rate is non-negotiable** — both LONG and SHORT windows must be computed.
4. **NO TRAFFIC ≠ GO** — empty sli pool means no signal, not healthy.
5. **Custom-SLI only (v1)** — fail fast if `customSli.indicator` is null.
6. **State which window triggered** — every verdict sentence must name both rates.
7. **Forecast is opt-in** — only run W-forecast when `--forecast` is passed.
8. **Filename = `SLO_BURN_*`** — never dt-rca or dt-rcf naming.

## Real-World Use Cases

1. **Pre-release gate** — "Should we ship 2.3.0 in the next 2 hours?" Run `--forecast`; if exhaustion projected within the release window, FREEZE.
2. **Mid-incident exec brief** — during active Davis problem, pull a one-page burn picture leadership can read.
3. **Weekly SRE review** — sweep top SLOs at `HORIZON:30d` for trending burn.
4. **Adaptive freeze trigger** — wire into CI; verdict ≠ GO blocks the deploy stage.
5. **Customer escalation defense** — customer claims breach; pull `-clean` briefing with sourced numbers.

## Substrate Notes (Validated 2026-05-28)

- SLOs are settings records, not Grail data objects
- `dtctl` envelope: parse with `parsed.get("result", parsed)`
- Indicator DQL must be executed via CLI timeframe flags, never inline
- v1 supports Custom SLI SLOs only
