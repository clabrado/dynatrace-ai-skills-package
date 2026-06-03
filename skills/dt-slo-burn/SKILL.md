---
name: dt-slo-burn
description: >-
  Error-Budget Burn Briefing for Dynatrace SLOs. Computes multi-window
  burn-rates per the Google SRE Workbook (fast + slow burn), identifies top
  contributing services / endpoints / users, surfaces correlated open Davis
  problems, and (optionally) forecasts budget-exhaustion datetime via
  predictive analytics. Produces a polished Markdown briefing (PDF optional via
  `--pdf`) with verdict ladder (GO / SLOW BURN / FAST BURN / FREEZE). Use when an SLO is degraded,
  a release is gated on error-budget posture, or a stakeholder needs a sourced
  burn briefing. Trigger: "SLO burn", "error budget", "burn rate",
  "is the SLO healthy", "should we ship", "release freeze", "budget
  exhaustion", "SLO forecast", "fast burn alert", "slow burn alert".
---

# dt-slo-burn — Dynatrace Error-Budget Burn Briefing

Agentic burn-rate analysis skill. Resolves an SLO, dispatches parallel workers
to compute multi-window burn-rates, top contributors, and correlated Davis
problems (optionally a budget-exhaustion forecast), synthesizes a verdict via
Davis CoPilot, and produces a complete briefing (Markdown by default; PDF
optional via `--pdf`).

This skill follows the **dt-rcf substrate pattern**: phased orchestration,
reference-driven workers, parallel dispatch in one message, absence-gate before
publishing, then synthesis → report. Do not deviate from the phase order.

---

## Usage

```
/dt-slo-burn SLO:"checkout-availability"                        # 1h horizon (default)
/dt-slo-burn SLO:"checkout-availability" HORIZON:6h             # 6h horizon
/dt-slo-burn SLO:"a3f2c8e1-..." HORIZON:24h --forecast          # 24h + exhaustion forecast
/dt-slo-burn SLO:"login-latency-99p" HORIZON:30d --forecast     # 30d compliance + forecast
/dt-slo-burn SLO:"checkout-availability" -clean                 # Sanitized briefing
/dt-slo-burn SLO:"checkout-availability" --pdf                  # Also render a PDF (default is MD-only)
```

## Arguments

- `SLO:"<id-or-name>"` — required anchor. Accepts either the SLO UUID or its display name.
- `HORIZON:1h|6h|24h|30d` — short-window horizon for burn evaluation. Default `1h`.
- `--forecast` — add Phase 1 worker `W-forecast` (budget-exhaustion datetime + confidence band).
- `--pdf` — also render a PDF alongside the Markdown. **Default is Markdown-only.** Accepts legacy alias `pdf=true`; `pdf=false` is a no-op (already the default).
- `-clean` — sanitize all identifying names (service, k8s namespace, user IDs, etc.) per dt-rca Phase 1.14 rules.
- `appendix=true` — full Appendix A query log; default OFF (one-line pointer).

> **Speed note:** run at **medium effort**. Quality is set by reference reads
> and query depth, not effort level.

---

## Sub-Skills Loaded Per Phase

dt-slo-burn carries NO forecast or service DQL beyond the orchestration
primitives below. Each worker reads ONE targeted reference at start, once, then
derives queries with dt-slo-burn scoping applied.

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-burn         | `~/.claude/skills/dt-obs-services/references/service-metrics.md` (RED metric patterns, error-rate + failure_count) |
| W-contributors | `~/.claude/skills/dt-obs-services/references/service-metrics.md` (group-by endpoint / k8s.workload / consumer dimensions) |
| W-davis        | `~/.claude/skills/dt-obs-problems/references/problem-correlation.md` (active-problem filter, scoped to affected entities) |
| W-forecast     | `~/.claude/skills/dt-obs-predictive-analytics/references/forecasting-analyzer.md` (timeseries-forecast tool reference, sizing rules) |

---

## ENTITY MODEL — SMARTSCAPE-NATIVE (NON-NEGOTIABLE)

Identical to dt-rcf. Span / log / event filtering MUST use
`dt.smartscape.service` + `smartscapeNodes`. NEVER `dt.entity.service == "..."`.
Read `~/.claude/skills/dt-rcf/SKILL.md` "ENTITY MODEL" for the canonical
patterns; everything there applies here unchanged.

**SLO-specific corollaries:**

- SLOs are tenant **settings resources**, NOT Grail data objects. There is no
  `fetch dt.slo` table. Resolve via `dtctl get slo --plain` (list) and
  `dtctl describe slo <id> -o json --plain` (detail). NEVER write `fetch dt.slo`.
- The SLI is a DQL expression embedded in the SLO record under
  `customSli.indicator` (note the lowercase `l` — `customSli`, NOT `customSLI`).
  Extract it **verbatim** in Phase 0b; do not reinvent or mutate it.
- Burn-rate math runs by **executing the SLI indicator over arbitrary time
  windows via dtctl flags** (`--default-timeframe-start` / `--default-timeframe-end`),
  NOT by injecting `from:/to:` into the DQL. The indicator already encodes its
  own Smartscape scope; the orchestrator only varies the timeframe.

**Known Limit (v1):** dt-slo-burn supports **Custom SLI SLOs only**. If the
SLO record has no `customSli.indicator` (template-based SLO), the run fails fast
per Phase 0b. Use the Dynatrace SLO app for template-based burn views, or
convert the SLO to a Custom SLI to use this skill.

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking — set up at the very start)

Maintain a per-run timing log next to the briefing: `LOG="SLO_BURN_{SLO_SLUG}_{DATE}.log"`.
At each phase boundary, append ONE backgrounded line:
```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line summary>" >> "$LOG"; } &
```
Log: START, Phase 0b complete (SLO summary), Phase 0b Step 4b Substrate Probe summary,
Phase 1 dispatch fan-out, Absence-Gate result, report written, RUN COMPLETE. Phase 6
prints the log path.

---

### Phase 0a: Argument Parser

```
ANCHOR (required):
  SLO:"<opaque-id>"   → SLO_ANCHOR_KIND = "id",   SLO_ANCHOR = id
  SLO:"<human-name>"  → SLO_ANCHOR_KIND = "name", SLO_ANCHOR = name
  (no SLO: token)     → FAIL with usage banner

  Detection heuristic: a value matching the tenant's opaque-id shape
  (UUID-like, or a long alnum/dash string with no spaces) is treated as
  KIND="id"; anything containing spaces or non-id characters is KIND="name".
  Phase 0b resolves either kind to a concrete SLO_ID.

FLAGS:
  HORIZON:1h|6h|24h|30d  → BURN_HORIZON (default 1h)
  --forecast             → FORECAST_MODE = true
  --pdf | pdf=true       → PDF_MODE = true (default FALSE — MD-only)
  pdf=false              → PDF_MODE = false (no-op; already the default)
  -clean                 → CLEAN_MODE = true
  appendix=true          → FULL_APPENDIX = true (default false)

DERIVED:
  SLO_SLUG               → safe filename slug from SLO_NAME (alnum + underscore)
                           after Phase 0b resolution; pre-resolution use first
                           8 chars of SLO_ANCHOR
  SHORT_WINDOW           → per BURN_HORIZON table below
  THRESHOLD              → per BURN_HORIZON table below
```

**Multi-window burn-rate table (Google SRE Workbook ch.5):**

| BURN_HORIZON | Long window | Short window | Threshold (×) | Budget burned if fired |
|--------------|-------------|--------------|---------------|------------------------|
| 1h           | 1h          | 5m           | 14.4          | ~5% in 1h              |
| 6h           | 6h          | 30m          | 6             | ~10% in 6h             |
| 24h          | 24h         | 2h           | 3             | ~10% in 24h            |
| 30d          | 30d         | 6h           | 1             | budget tracking only   |

Always evaluate BOTH the long-window and the short-window rate. The alert is
the AND of both crossing threshold — short alone is noise, long alone is
stale. Report the verdict per the rules in Phase 2.

VALIDATION: if no SLO: anchor → print usage and exit.

---

### Phase 0-auth: Pre-warm dtctl session (Orchestrator — proactive step 0)

**FIRST action of the run — before any query, before any dispatch:**
```
dtctl auth refresh
```
Silent refresh via on-disk token store (`DTCTL_TOKEN_STORAGE=file`, harness env, never the
Keychain). If it fails (no/expired refresh token — first-ever use or weeks idle), run ONCE:
`dtctl auth login --plain --safety-level readonly`. **Workers never authenticate** — they
inherit the pre-warmed on-disk token. If a worker returns `<gap: auth>`, the orchestrator
refreshes once and re-dispatches only that worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

This is identical to dt-rcf Phase 0-auth — read it there for full rationale.

---

### Phase 0b: SLO Bootstrap (Sequential — Orchestrator)

Build the **SLOContext** that all Phase 1 workers receive. SLOs are **settings
resources** — resolved via `dtctl get`/`describe`, never via Grail `fetch`.

#### Step 1 — Resolve SLO_ID

**If SLO_ANCHOR_KIND == "id":** skip directly to Step 2 with `SLO_ID = SLO_ANCHOR`.

**If SLO_ANCHOR_KIND == "name":** name-based `dtctl describe slo` returns
`400 "Could not decode ID"`. Resolve to an opaque ID via the list endpoint:

```
dtctl get slo --plain
```

Returns JSON of shape (handle BOTH envelope shapes — TTY wrapping vs subprocess):
```python
parsed = json.loads(stdout)
records = parsed.get("result", parsed)  # 'result' wrapper present under TTY/agent
# records is a list of { id, name, description, version, criteria, tags }
```

Resolution rules (in order — first hit wins):
1. **Exact match** on `name` field (case-sensitive). >1 exact match → FAIL with
   the disambiguation list (`id  |  name  |  description`) and instruct the user
   to re-run with `SLO:"<opaque-id>"`.
2. **Case-insensitive substring** match on `name`. Exactly one hit → use it.
   Zero or >1 hits → FAIL with the candidate list (limit 10) and instruct the
   user to re-run with a more specific name or an opaque id.

On any failure, emit the **SLO Not Found** banner (see ERROR HANDLING) and exit.

#### Step 2 — Fetch SLO detail

```
dtctl describe slo {SLO_ID} -o json --plain
```

Parse with the same envelope-tolerant pattern:
```python
parsed  = json.loads(stdout)
record  = parsed.get("result", parsed)
```

Extract these fields off `record`:

```
SLO_ID                   record["id"]
SLO_NAME                 record["name"]
SLO_DESCRIPTION          record.get("description", "")
SLO_TARGET_PCT           record["criteria"][0]["target"]      # e.g. 98.0
SLO_WARNING_PCT          record["criteria"][0].get("warning") # optional, may be null
SLO_EVAL_WINDOW          record["criteria"][0]["timeframe"]   # e.g. "-30d"
CUSTOM_SLI               record.get("customSli")              # NOTE: lowercase 'l'
SLI_INDICATOR_DQL        CUSTOM_SLI["indicator"] if CUSTOM_SLI else None
TAGS                     record.get("tags", [])
```

#### Step 3 — Custom-SLI gate (HARD FAIL for template SLOs)

```python
if not SLI_INDICATOR_DQL:
    emit_banner_and_exit("""
/dt-slo-burn v1 supports SLOs with a Custom SLI indicator only.
The SLO "{name}" is template-based. Use the Dynatrace SLO app for template-based burn views,
or convert this SLO to a Custom SLI to use /dt-slo-burn.
""".format(name=SLO_NAME))
```

No SLI indicator means no DQL to execute over arbitrary windows. Template SLOs
are intentionally out of scope for v1 — do not attempt to synthesize an SLI.

#### Step 4 — Derive timeframe ISO strings

The orchestrator computes the timeframe pairs that workers will pass to
`dtctl query --default-timeframe-start/--default-timeframe-end`. **Do not
mutate the SLI indicator DQL** — vary timeframe via flags only.

| BURN_HORIZON | LONG window (start..end) | SHORT window (start..end) | FORECAST training window |
|---|---|---|---|
| 1h  | now-1h..now  | now-5m..now  | now-6h..now  |
| 6h  | now-6h..now  | now-30m..now | now-24h..now |
| 24h | now-24h..now | now-2h..now  | now-3d..now  |
| 30d | now-30d..now | now-6h..now  | now-60d..now |

For HORIZON 24h/30d, also compute the 1h/5m fast-burn pair so the briefing can
report both fast and slow burn rates when relevant.

#### Step 4b — Substrate Probe (probe-then-query — run ONCE, before trusting any literal)

> Runs after Step 4 (it needs `SLI_INDICATOR_DQL` from Step 2 and the LONG-window
> ISO pair from Step 4) and before Step 5 stores the SLOContext.

The downstream workers (W-burn, W-davis) survive field/schema drift because
they are reference-driven. The Phase 0b bootstrap historically hardcoded the
SLI-pool assumption and the `dt.davis.problems` correlation-field syntax — both
have drifted on live tenants. **Probe the real shape at runtime and bind it into
variables the later queries interpolate**, rather than assuming a stale literal.
Keep it lightweight — 2 cheap queries.

**Probe A — SLI pool is non-null over the burn window (so empty = probed NO_TRAFFIC, not assumed).**
Execute `SLI_INDICATOR_DQL` verbatim over the LONG-window ISO pair from Step 4
(via `--default-timeframe-start/--default-timeframe-end`; **ISO-8601 only** —
relative `now-Nd` is rejected by these flags). Flatten the `sli` arrays, drop
nulls, pool across records:
```
SLI_POOL_SIZE = count(non-null sli values across all records)
```
- `SLI_POOL_SIZE > 0` → bind `SLI_PROBE = "live"`; W-burn proceeds normally. An
  empty pool in any *later* worker window is then a genuine, probed NO_TRAFFIC.
- `SLI_POOL_SIZE == 0` over the LONG window → before concluding NO_TRAFFIC,
  re-run over a 2× wider window (the Phase 1.5 widen). Still empty → bind
  `SLI_PROBE = "no_traffic"` and carry it into Phase 1.5 so the NO TRAFFIC
  verdict is gated by proof, never assumed. This is the orchestrator-side
  pre-confirmation of the absence the worker would otherwise only hypothesize.

**Probe B — Davis correlation fields + the working membership syntax.**
W-davis correlates open problems to the SLO's service entities via two
candidate array fields. Both coexist on this tenant but the *membership
operator* matters — confirm presence AND bind the syntax:
```dql
fetch dt.davis.problems, from:now()-24h
| filter not(dt.davis.is_duplicate)
| fieldsAdd has_smartscape = isNotNull(`smartscape.affected_entity.ids`)
| fieldsAdd has_classic    = isNotNull(affected_entity_ids)
| summarize total = count(),
            smartscape_pop = countIf(has_smartscape == true),
            classic_pop    = countIf(has_classic == true)
```
- Bind `DAVIS_FIELDS` from a **CANDIDATE LIST** ordered
  `[ ("smartscape.affected_entity.ids", "affected_entity_ids")  /* corrected-2026-06-03 */ ]`.
  Both populated (typical: classic ~100%, smartscape a subset) → keep the
  dual-model OR in W-davis. If exactly one is populated, the OR still works —
  the empty side simply never matches.
- **Membership syntax (HARD — validated 2026-06-03):** the dotted field is only
  filterable through the **function form** ``in("{SVC_ID}", `smartscape.affected_entity.ids`)``.
  The infix form (`"{SVC_ID}" in smartscape.affected_entity.ids`) **fails
  `PARSE_ERROR` (`isn't allowed here`)** on this tenant. W-davis uses the
  function form; do NOT regress to infix. Backtick the dotted field name.
- If NEITHER candidate field exists (schema fully drifted) → W-davis emits its
  "correlation unavailable" gap note rather than guessing — absence stays gated.

**Emit a one-line probe summary to the run log**, e.g.:
```
Probe: SLI pool=615 (live) over LONG window | davis fields → classic 554/554, smartscape 167/554 | membership=in(needle,`field`)
```

#### Step 5 — Store SLOContext JSON

```json
{
  "SLO_ID": "...",
  "SLO_NAME": "...",
  "SLO_DESCRIPTION": "...",
  "SLI_INDICATOR_DQL": "<verbatim DQL from customSli.indicator>",
  "SLO_TARGET_PCT": 98.0,
  "SLO_BUDGET_PCT": 2.0,
  "SLO_WARNING_PCT": 99.0,
  "SLO_EVAL_WINDOW": "-30d",
  "BURN_HORIZON": "1h",
  "SHORT_WINDOW": "5m",
  "THRESHOLD": 14.4,
  "LONG_WINDOW_FROM_ISO": "2026-05-28T09:00:00Z",
  "LONG_WINDOW_TO_ISO":   "2026-05-28T10:00:00Z",
  "SHORT_WINDOW_FROM_ISO":"2026-05-28T09:55:00Z",
  "SHORT_WINDOW_TO_ISO":  "2026-05-28T10:00:00Z",
  "FORECAST_TRAINING_FROM_ISO": "2026-05-28T04:00:00Z",
  "FORECAST_TRAINING_TO_ISO":   "2026-05-28T10:00:00Z",
  "TAGS": [...],
  "FORECAST_MODE": false,
  "PDF_MODE": false,
  "CLEAN_MODE": false,
  "SLI_PROBE": "live",
  "SLI_POOL_SIZE": 615,
  "DAVIS_FIELDS": ["smartscape.affected_entity.ids", "affected_entity_ids"],
  "DAVIS_MEMBERSHIP": "function"
}
```

`SLI_PROBE`, `SLI_POOL_SIZE`, `DAVIS_FIELDS`, and `DAVIS_MEMBERSHIP` are bound by
the Step 4b Substrate Probe and passed to W-burn / W-davis so they query the
confirmed substrate instead of a hardcoded assumption.

Where `SLO_BUDGET_PCT = 100 - SLO_TARGET_PCT` — the error-budget percentage used
as the denominator in burn-rate math.

---

### Phase 0c: Reference Strategy

dt-slo-burn is **reference-driven** — workers carry no hardcoded DQL beyond the
burn-rate primitive below. Each worker reads ONE targeted reference at start,
once, derives query patterns from it, and applies dt-slo-burn scoping (SLI
expression, SLO_FILTER, time windows). This is the correctness design: SLI
field names and metric paths stay current automatically.

**Orchestrator exceptions (inline only):** Phase 0b SLO resolution and the
Phase 1.5 Absence-Gate confirmation queries remain inline — they are
orchestration primitives.

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

**Dispatch ALL workers in ONE message with multiple Agent calls — true concurrency.** Sending
them one-at-a-time defeats the architecture. Workers W-burn, W-contributors, W-davis run
concurrently. Add W-forecast only if `FORECAST_MODE == true`.

Total worker count: **3 always, 4 if `--forecast`**.

All workers run as `subagent_type: Explore`. Worker prompt template, hygiene rules, and
PhaseResult shape follow dt-rcf Phase 1 verbatim — read
`~/.claude/skills/dt-rcf/SKILL.md` "Worker Prompt Template" for the full text. The
dt-slo-burn–specific worker bodies follow.

**Prepend this to EVERY worker prompt as its FIRST content (before "STEP 1 — READ YOUR REFERENCE"):**

> ANTI-FABRICATION (READ FIRST — non-negotiable):
> If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
> empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
> gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
> did not read directly from a successful query result. Inventing plausible-looking numbers is the
> single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
> is never acceptable. If you cannot point to the query result a number came from, do not emit it.

---

#### W-burn — Multi-window Burn-Rate

**INDICATOR-DRIVEN — execute the SLO's `SLI_INDICATOR_DQL` verbatim** over each
burn window via `dtctl query` timeframe flags. Do NOT mutate the DQL. Do NOT
inject `from:` / `to:`. Do NOT consult `service-metrics.md` — the SLO record
is the authority for what "good" and "valid" mean.

> The Phase 0b Step 4b probe already confirmed the indicator returns a non-null
> `sli` pool over the LONG window (`SLI_PROBE == "live"`, pool `SLI_POOL_SIZE`).
> So an empty pool in any window here is a real signal, not a broken query — emit
> `window_sli == None` and let Phase 1.5 confirm NO_TRAFFIC. If the orchestrator
> bound `SLI_PROBE == "no_traffic"`, the substrate was already pre-confirmed empty.

The indicator DQL looks like this (sample shape — actual text varies per SLO):
```
timeseries { total=sum(dt.service.request.count), failures=sum(dt.service.request.failure_count) }, by:{dt.entity.service}, filter:{in(dt.entity.service, {"SERVICE-..."})}
| fieldsAdd sli=(((total[]-failures[])/total[])*(100))
| fieldsRemove total, failures
```
Each output record carries an `sli` array — one value per timeseries interval.

**Burn-rate definition (Google SRE Workbook ch.5):**
```
budget         = 100 - SLO_TARGET_PCT
window_sli     = mean(non_null(flatten(sli arrays across records)))
error_rate_pct = 100 - window_sli
burn_rate      = error_rate_pct / budget
```
`burn_rate = 1.0` means the SLO is burning at the exact rate that would exhaust
the budget over the eval window. `14.4×` means 1/14.4 of the eval window suffices
to consume the full budget (~5% in 1h on a 30d window).

**Execution pattern (run BOTH calls in one parallel batch):**

For each (window_from, window_to) pair derived in Phase 0b, invoke:
```bash
dtctl query '<SLI_INDICATOR_DQL verbatim, single-quoted>' \
  --default-timeframe-start '<ISO start>' \
  --default-timeframe-end   '<ISO end>' \
  -o json --plain
```

Parse with the envelope-tolerant pattern (`records = parsed.get("result", parsed)`).
For each record, flatten the `sli` array, drop nulls, and pool across records:

```python
import statistics
pool = []
for rec in records:
    for v in rec.get("sli", []):
        if v is not None:
            pool.append(float(v))
if not pool:
    window_sli = None       # → NO TRAFFIC for this window (see Phase 1.5)
else:
    window_sli = statistics.fmean(pool)
error_rate_pct = None if window_sli is None else (100.0 - window_sli)
burn_rate      = None if error_rate_pct is None else (error_rate_pct / SLO_BUDGET_PCT)
```

**Required calls per HORIZON:**

| BURN_HORIZON | Calls made | Rates emitted |
|---|---|---|
| 1h  | LONG=1h, SHORT=5m | `fast_long_burn`, `fast_short_burn` |
| 6h  | LONG=6h, SHORT=30m | `slow_long_burn`, `slow_short_burn` |
| 24h | LONG=24h, SHORT=2h + extra 1h/5m pair | both fast and slow pairs |
| 30d | LONG=30d, SHORT=6h + extra 1h/5m + 6h/30m pairs | all pairs |

For HORIZON 1h and 6h, only the matching pair is computed. For 24h and 30d,
compute the additional fast/slow pairs so the briefing can call out either burn
class.

**Verdict logic (compute in worker, report in PhaseResult):**

Per Google SRE Workbook ch.5 multi-window primary criterion, an alert requires
**BOTH** the long and short burn rates to cross threshold simultaneously:

- **Fast burn (1h/5m, threshold 14.4):**
  fires only if `1h_burn ≥ 14.4 AND 5m_burn ≥ 14.4`
- **Slow burn (6h/30m, threshold 6):**
  fires only if `6h_burn ≥ 6 AND 30m_burn ≥ 6`
- **24h pair (24h/2h, threshold 3):** fires only if both cross.
- **30d pair (30d/6h, threshold 1):** budget tracking.

State derivation (per applicable pair):
- both ≥ threshold → `FIRING`
- long ≥, short <  → `RECOVERING`
- long <, short ≥  → `SPIKE_ONLY`
- both <            → `OK`
- either window has `window_sli == None` (no events) → `NO_TRAFFIC`

The overall `alert_state` is the highest-severity state across all evaluated pairs
(NO_TRAFFIC > FIRING > RECOVERING > SPIKE_ONLY > OK for verdict-laddering purposes;
NO_TRAFFIC short-circuits to the NO TRAFFIC banner in Phase 1.5).

**PhaseResult shape:**
```json
{
  "horizon": "1h",
  "threshold_x": 14.4,
  "pairs": [
    {
      "class": "fast",
      "long_window": "1h",  "short_window": "5m",  "threshold": 14.4,
      "long_burn_rate": 18.2, "short_burn_rate": 22.7,
      "long_error_rate_pct": 0.91, "short_error_rate_pct": 1.13,
      "long_window_sli": 99.09, "short_window_sli": 98.87,
      "long_window_from_iso": "...", "long_window_to_iso": "...",
      "short_window_from_iso": "...", "short_window_to_iso": "...",
      "state": "FIRING|RECOVERING|SPIKE_ONLY|OK|NO_TRAFFIC"
    }
  ],
  "alert_state": "FIRING|RECOVERING|SPIKE_ONLY|OK|NO_TRAFFIC",
  "budget_consumed_pct_in_long_window": 6.3,
  "indicator_dql_executed": "<verbatim>",
  "gaps": []
}
```

---

#### W-contributors — Top Contributors

**REFERENCE-DRIVEN — read `~/.claude/skills/dt-obs-services/references/service-metrics.md`
ONCE** (› "Failure Analysis", "k8s.workload group-by" examples). Apply dt-slo-burn scoping.

**dt-slo-burn scoping for EVERY query:** the same `SLO_FILTER` Smartscape scope; spans use
`start_time`; alias bins; single-quote the dtctl arg. Look at the BURN_HORIZON window
(not the eval window) — contributors to CURRENT burn, not historical share.

**Queries to run (ALL in one parallel batch — 3 queries):**

1. **Top endpoints by failure count.**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:now()-{BURN_HORIZON}
   | filter dt.smartscape.service in [
       smartscapeNodes SERVICE
       | filter id == "{SLO_FILTER_SERVICE_ID}"
       | fields id
     ]
   | filter request.is_root_span == true
   | filter request.is_failed == true
   | summarize fails = count(), by: { endpoint.name, http.response.status_code }
   | sort fails desc
   | limit 10
   ```

2. **Top callees / downstream dependencies driving failures.**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:now()-{BURN_HORIZON}
   | filter dt.smartscape.service in [
       smartscapeNodes SERVICE
       | filter id == "{SLO_FILTER_SERVICE_ID}"
       | fields id
     ]
   | filter span.kind == "client"
   | filter request.is_failed == true
   | fieldsAdd callee_name = getNodeName(dt.smartscape.service)
   | summarize fails = count(), by: { callee_name, span.name }
   | sort fails desc
   | limit 10
   ```

3. **Top consumer / caller dimensions (user-agent, request_attribute.user, or
   k8s.workload of caller) — discover schema from a sample span first if unknown.**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:now()-{BURN_HORIZON}
   | filter dt.smartscape.service in [
       smartscapeNodes SERVICE
       | filter id == "{SLO_FILTER_SERVICE_ID}"
       | fields id
     ]
   | filter request.is_root_span == true
   | filter request.is_failed == true
   | summarize fails = count(),
       by: { http.user_agent, k8s.workload.name }
   | sort fails desc
   | limit 10
   ```

**PhaseResult shape:**
```json
{
  "top_endpoints":  [{ "endpoint": "...", "status_code": 500, "fails": 0 }],
  "top_callees":    [{ "callee": "...", "span_name": "...", "fails": 0 }],
  "top_consumers":  [{ "user_agent": "...", "workload": "...", "fails": 0 }],
  "concentration_note": "85% of failures concentrated on 2 endpoints | broad-base failures across N endpoints",
  "gaps": []
}
```

---

#### W-davis — Active Davis Problems on SLO-affected Entities

**REFERENCE-DRIVEN — read `~/.claude/skills/dt-obs-problems/references/problem-correlation.md`
ONCE** (› active-problem filter pattern, scoped filter on affected entities). Apply
dt-slo-burn scoping.

**dt-slo-burn scoping:**
- ALWAYS `not(dt.davis.is_duplicate)` — required by RULE on every dt.davis.problems query.
- `event.status == "ACTIVE"` to capture currently firing problems.
- Filter to the SLO_FILTER service entity using the dual-model OR pattern (Smartscape +
  classic), since both `affected_entity_ids` and `smartscape.affected_entity.ids` coexist
  (confirmed by the Phase 0b Step 4b probe → `DAVIS_FIELDS`).
- **Membership uses the function form `in(needle, field)`, NOT infix.** The dotted
  Smartscape field must be backtick-quoted, and the infix form
  (`"{ID}" in smartscape.affected_entity.ids`) **fails `PARSE_ERROR` on this
  tenant** (validated 2026-06-03 via `DAVIS_MEMBERSHIP="function"`). Never regress to infix.

**Queries to run (1 query — ALL active problems touching the SLO entities):**

<!-- VALIDATED 2026-06-03: function-form membership returns rows; infix form parse-errors -->
```dql
fetch dt.davis.problems, from:now()-{BURN_HORIZON}
| filter not(dt.davis.is_duplicate) and event.status == "ACTIVE"
| filter in("{SLO_FILTER_SERVICE_ID}", `smartscape.affected_entity.ids`)
      or in("{SLO_FILTER_SERVICE_ID}", affected_entity_ids)
| fieldsAdd duration_min = (coalesce(event.end, now()) - event.start) / 60e9
| sort event.start asc
| limit 20
| fields display_id, event.name, event.category, event.start, duration_min,
         root_cause_entity_name, affected_entity_ids
```

**PhaseResult shape:**
```json
{
  "active_problem_count": 0,
  "active_problems": [{
    "display_id": "P-...",
    "title": "...",
    "category": "error|slowdown|resource|availability",
    "start": "ISO",
    "duration_min": 0,
    "root_cause": "..."
  }],
  "correlation_note": "1 active problem matches the failing endpoint | no active problems — burn likely caused by load/customer",
  "gaps": []
}
```

---

#### W-forecast (Conditional — only if `FORECAST_MODE == true`)

**REFERENCE-DRIVEN — read
`~/.claude/skills/dt-obs-predictive-analytics/references/forecasting-analyzer.md` ONCE**
(› "Tool Reference: timeseries-forecast", "Interval / Horizon Sizing"). Use the
`timeseries-forecast` tool to project burn-rate forward.

**Sizing rules (from forecasting-analyzer.md):**

| BURN_HORIZON | timeseries interval | forecastHorizon (steps) | Training window |
|--------------|---------------------|-------------------------|-----------------|
| 1h           | 1m                  | 60   (= 1h ahead)       | now-6h          |
| 6h           | 5m                  | 72   (= 6h ahead)       | now-24h         |
| 24h          | 15m                 | 96   (= 24h ahead)      | now-3d          |
| 30d          | 1h                  | 168  (= 7d ahead)       | now-60d         |

Training history must be at least 2× the forecast horizon AND at least 14 non-null values
in the last third of the training window. If `SLO_HISTORY_DAYS` is below the training
window length, the forecast will still run but its confidence band will be wide —
already flagged in Phase 0b.

**Build the training series by executing the SLI indicator over the forecast
training window — do NOT rewrite the indicator into a separate timeseries query.**

```bash
dtctl query '<SLI_INDICATOR_DQL verbatim>' \
  --default-timeframe-start '{FORECAST_TRAINING_FROM_ISO}' \
  --default-timeframe-end   '{FORECAST_TRAINING_TO_ISO}' \
  -o json --plain
```

From each record, flatten the `sli` array against its timestamp array (the
indicator's `timeseries` produces aligned arrays). Convert `sli` → `burn_rate`
elementwise: `burn_rate[i] = (100 - sli[i]) / SLO_BUDGET_PCT`. Pool across
records if the indicator groups by service; otherwise use the single series.

Pass the resulting (timestamp[], burn_rate[]) series as the input to
`mcp__dynatrace__execute_timeseries_forecast` with `forecastHorizon = {steps}`
and `generalParameters.timeframe.startTime = {FORECAST_TRAINING_FROM_ISO}`.

If the SLI indicator's natural interval is finer/coarser than the
`FORECAST_INTERVAL` from the sizing table, downsample (mean over bin) or accept
the indicator's native interval — never alter the indicator DQL itself.

**Project budget exhaustion:**

Given the SLO target T and eval window W:
```
budget_total_bad_allowed = (1 - T/100) * valid_events_over_W
budget_consumed_so_far   = sum(bad_events_so_far_in_W)
budget_remaining         = budget_total_bad_allowed - budget_consumed_so_far
projected_bad_per_step   = mean(point_forecast_array)
projected_steps_to_exhaust = budget_remaining / projected_bad_per_step
projected_exhaust_datetime = now() + projected_steps_to_exhaust * interval
```

**PhaseResult shape:**
```json
{
  "forecast_horizon": "1h",
  "forecast_interval": "1m",
  "point_forecast_burn_rate": [/* array */],
  "lower_band": [/* array */],
  "upper_band": [/* array */],
  "budget_remaining_pct": 42.0,
  "projected_exhaust_datetime": "2026-05-29T03:22:00Z",
  "projected_exhaust_datetime_lower": "2026-05-29T01:10:00Z",
  "projected_exhaust_datetime_upper": "2026-05-29T08:55:00Z",
  "trend": "🔴 accelerating | 🟠 elevated | 🟢 stable | 🔵 declining",
  "confidence_note": "wide band (history <7d) | narrow band (14d+ history)",
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator — run before trusting any "absent" finding)

Before using any worker result, scan every PhaseResult for absence claims ("no
failures in window", "no active problems", "no contributors", "forecast unavailable",
"SLI returned 0 events"). For EACH such claim:

1. Require the worker's **proof** (the indicator output sample + the parsed pool size).
   If the worker did not attach it, the claim is invalid.
2. Run ONE cheap confirmation query yourself. Examples:
   - W-burn reports `window_sli == None` (empty `sli` pool) → re-run the SLI
     indicator over a 2× wider window via `dtctl query` with widened
     `--default-timeframe-*` flags. If the pool is still empty, the SLO truly
     had no qualifying events — emit the NO TRAFFIC banner.
   - W-davis returns 0 active problems → run an unfiltered `fetch dt.davis.problems`
     over 24h. If still zero, the absence is real.
3. Only after a confirmation query genuinely returns zero across the broad scope may
   "absent" enter the briefing. This is the binding application of the same rule that
   gates dt-rcf — a worker's empty result is a hypothesis to verify, never a conclusion
   to publish.

**Special SLO-specific case:** if W-burn reports `alert_state == "NO_TRAFFIC"`
(both long and short window SLI pools empty), the briefing's verdict is
`NO TRAFFIC` (not `OK`) — emit a banner explaining the SLO had no qualifying
events to evaluate. Continue to W-davis + W-contributors context but skip
burn-rate tables.

---

### Phase 2: Synthesis & Verdict Ladder (Orchestrator, Sequential)

After all workers return (and Absence Gate has cleared), synthesize the **Verdict** using
this ladder. State explicitly **which window triggered**.

```
Inputs: W-burn.alert_state, W-burn.pairs[*].long_burn_rate / short_burn_rate
        (use the pair that matches BURN_HORIZON's primary class — fast for 1h,
        slow for 6h, both for 24h/30d),
        W-forecast.budget_remaining_pct (if FORECAST_MODE)

LADDER:
  alert_state == "NO_TRAFFIC"
    → VERDICT = "NO TRAFFIC"
    → SLI denominator is zero across both windows; SLO has no signal
    → skip burn-rate table; cite indicator + window in prose

  alert_state == "FIRING" AND BURN_HORIZON in (1h, 6h)
    → VERDICT = "FAST BURN / FREEZE"
    → recommend release freeze; both windows over threshold
    → cite both rates explicitly

  alert_state == "FIRING" AND BURN_HORIZON in (24h, 30d)
    → VERDICT = "SLOW BURN"
    → no immediate freeze, but budget being consumed faster than sustainable
    → cite both rates explicitly

  alert_state == "RECOVERING"
    → VERDICT = "SLOW BURN (recovering)"
    → long window over, short window below; incident may be resolving
    → recommend continued monitoring, no freeze

  alert_state == "SPIKE_ONLY"
    → VERDICT = "SPIKE — MONITORING"
    → short window over, long window below; transient
    → recommend monitoring; do not freeze on a 5m spike alone

  alert_state == "OK"
    → VERDICT = "GO"
    → both windows below threshold; SLO healthy
```

**Forecast overlay (if FORECAST_MODE):**
- If `projected_exhaust_datetime` falls within the BURN_HORIZON → escalate verdict by one
  step (GO → SLOW BURN, SLOW BURN → FAST BURN / FREEZE).
- If `projected_exhaust_datetime` is null (forecast failed / insufficient history) → note
  in the verdict prose; do NOT escalate.

**Cross-reference Davis problems:**
- W-davis returned ≥1 active problem matching the top failing endpoint → the briefing's
  remediation should reference that problem ID (deep-link to it).
- W-davis returned 0 active problems but burn is FIRING → flag as "Davis hasn't fired
  yet — burn detected via SLO before problem opens." This is a feature, not a bug.

Pass the verdict + supporting evidence to Phase 3.

---

### Phase 3: Davis CoPilot Synthesis (Orchestrator, Sequential)

Pass the **verdict + top contributors + forecast** to Davis CoPilot for remediation
guidance. Do NOT pass raw worker output.

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
SLO BURN BRIEFING: {SLO_NAME} | {BURN_HORIZON} horizon
VERDICT: {VERDICT} ({alert_state}, triggered by {which window})

CURRENT BURN RATES (per evaluated pair):
{for each pair in W-burn.pairs:}
- {pair.class} pair — long {pair.long_window} {pair.long_burn_rate}× /
                       short {pair.short_window} {pair.short_burn_rate}×
                       (threshold {pair.threshold}×, state {pair.state})
- Budget consumed in primary long window: {budget_consumed_pct_in_long_window}%

TOP CONTRIBUTORS:
- Endpoints: {top 3 endpoint:status:count rows}
- Callees:   {top 3 callee:span:count rows}
- Consumers: {top 3 user_agent:workload:count rows}

ACTIVE DAVIS PROBLEMS ON SLO ENTITIES: {N}
{list display_id + title for each}

FORECAST (if FORECAST_MODE):
- Projected budget exhaustion: {datetime} (range: {lower} – {upper})
- Trend: {accelerating | stable | declining}
- Confidence: {wide | narrow} band

QUESTIONS:
1. Given the verdict and contributor concentration, what is the most likely root cause?
2. For the top failing endpoint+status combination, what specific remediation steps?
3. Should this trigger an immediate freeze, or is monitoring sufficient?
4. What additional signal would increase confidence in the verdict?
```

Davis CoPilot response feeds the "Recommended Actions" section of the briefing.

---

### Phase 4: Report Generation

If CLEAN_MODE, build the sanitization map first (identical to dt-rca Phase 1.14 — read
that section for full rules including the entity-name dictionary, k8s namespace remapping,
and consistency enforcement) and compose with sanitized names from the start.

**Emit the briefing EXACTLY in the section order below — canonical skeleton.**

```markdown
# dt-slo-burn Error-Budget Burn Briefing
## SLO: {SLO_NAME} ({BURN_HORIZON} Horizon) — {VERDICT}

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**SLO Anchor:** {SLO_ANCHOR}  |  **Target:** {SLO_TARGET_PCT}%  |  **Eval Window:** {SLO_EVAL_WINDOW}  |  **Burn Horizon:** {BURN_HORIZON}

---

## Executive Verdict

**{VERDICT}** — {one-sentence reason citing which window triggered and the burn-rate magnitude.}

[Example: "FAST BURN / FREEZE — both 1h (18.2×) and 5m (22.7×) windows are over the
14.4× fast-burn threshold; ~6.3% of the 30-day error budget consumed in the last hour."]

[If FORECAST_MODE:]
> **Forecast:** Budget exhaustion projected at **{projected_exhaust_datetime}** (range:
> {lower} – {upper}). Trend: {trend}. Confidence: {confidence_note}.

---

## Burn-Rate Table

[Skip this section entirely if VERDICT == "NO TRAFFIC".]

| Pair | Long Window Rate | Short Window Rate | Threshold | State |
|------|------------------|-------------------|-----------|-------|
[One row per W-burn.pairs entry:]
| {pair.class} ({pair.long_window}/{pair.short_window}) | {pair.long_burn_rate}× | {pair.short_burn_rate}× | {pair.threshold}× | {state with emoji: 🔴 FIRING / 🟠 RECOVERING / 🟡 SPIKE_ONLY / 🟢 OK} |

**Verdict mapping:** Both rates over → FAST BURN / FREEZE (fast pair) or SLOW BURN
(slow pair). Long-only over → RECOVERING. Short-only over → SPIKE — monitoring. Both
under → GO. Empty SLI pool → NO TRAFFIC (table omitted).

**Budget consumed in primary long window:** {budget_consumed_pct_in_long_window}%

---

## Top Contributors

### By Endpoint
| Endpoint | Status | Failures | % of Total |
|----------|--------|----------|------------|
[Top 5–10 rows from W-contributors.top_endpoints]

### By Downstream Dependency
| Callee | Operation | Failures |
|--------|-----------|----------|
[Top 5 rows from W-contributors.top_callees]

### By Consumer
| User-Agent / Workload | Failures |
|-----------------------|----------|
[Top 5 rows from W-contributors.top_consumers]

**Concentration:** {concentration_note from W-contributors}

---

## Forecast

[Include ONLY if FORECAST_MODE. Else: "Forecast not requested — re-run with `--forecast` to project budget exhaustion."]

| Field | Value |
|-------|-------|
| Forecast Horizon | {forecast_horizon} |
| Interval | {forecast_interval} |
| Projected Budget Exhaustion | {projected_exhaust_datetime} |
| Confidence Range | {lower} – {upper} |
| Trend | {trend} |
| Budget Remaining | {budget_remaining_pct}% |
| Confidence Note | {confidence_note} |

[Optional: a Mermaid line chart showing burn_rate history + forecast band, per dt-rca
diagram sizing rules. ASCII-only in PDF.]

---

## Davis Synthesis

### Correlated Active Problems
| Display ID | Title | Category | Duration | Root Cause |
|------------|-------|----------|----------|------------|
[Rows from W-davis.active_problems]

[If 0 active problems and burn FIRING: callout "Davis hasn't fired yet — burn detected
via SLO before problem opens. Consider whether a Davis detection rule is missing."]

### Davis CoPilot Assessment
[Davis CoPilot response from Phase 3, formatted as prose + bullets.]

---

## Recommended Actions

| Priority | Action | Owner | Urgency |
|----------|--------|-------|---------|
[Derived from Davis CoPilot + verdict ladder. P0/P1/P2.]

**Decision guidance:**
- **VERDICT = FAST BURN / FREEZE** → halt deploys to the affected service; engage on-call.
- **VERDICT = SLOW BURN** → no freeze, but escalate to service owner; investigate within 24h.
- **VERDICT = SLOW BURN (recovering)** → continue monitoring; confirm short-window stays low.
- **VERDICT = SPIKE — MONITORING** → review the 5m spike for legitimacy; no action if isolated.
- **VERDICT = GO** → SLO healthy; release as planned.

---

## Appendix A: Briefing Details

| Field | Value |
|-------|-------|
| SLO ID | {SLO_ID} |
| SLO Name | {SLO_NAME} |
| SLI Indicator (Custom SLI DQL) | `{SLI_INDICATOR_DQL}` |
| Target | {SLO_TARGET_PCT}% |
| Eval Window | {SLO_EVAL_WINDOW} |
| Burn Horizon | {BURN_HORIZON} (long), {SHORT_WINDOW} (short) |
| Threshold | {THRESHOLD}× |
| Workers Dispatched | W-burn, W-contributors, W-davis {+ W-forecast if FORECAST_MODE} |
| Davis CoPilot | {Used / Unavailable} |
| Active Problems | {count} |
| Forecast Mode | {FORECAST_MODE} |
| History Days | {SLO_HISTORY_DAYS} |
| Substrate Probe | SLI pool {SLI_POOL_SIZE} ({SLI_PROBE}); davis fields {DAVIS_FIELDS}; membership {DAVIS_MEMBERSHIP} |

### dt-slo-burn Telemetry
| Worker | Sub-Skill | Key Findings | Gaps |
|--------|-----------|--------------|------|
[W-burn / W-contributors / W-davis / W-forecast rows]

### Queries Run
[If FULL_APPENDIX == true: full DQL log of every query, with timing.
 If FULL_APPENDIX == false: one line → "Full query log: {LOG_PATH}".]

---

**Links:**
- [View SLO](https://{TENANT}.apps.dynatrace.com/ui/apps/dynatrace.slo/slo/{SLO_ID})
- [View Service](https://{TENANT}.apps.dynatrace.com/ui/entity/{SLO_FILTER_SERVICE_ID})

---

*End of Briefing*
```

### Pre-Save Self-Check (run before Phase 5 writes anything)

If any item is "no", fix before writing:
- [ ] H1 title + `## SLO:` H2 subtitle present with VERDICT in subtitle
- [ ] Header is `Generated / Analyst: Claude {MODEL} / Environment` (no tool-version banner)
- [ ] Executive Verdict is exactly one sentence + optional forecast banner — NOT a paragraph
- [ ] Burn-Rate Table present with BOTH windows on separate rows
- [ ] Top Contributors has all three subsections (Endpoint / Dependency / Consumer)
- [ ] Forecast section present if FORECAST_MODE, otherwise the explicit "not requested" line
- [ ] Davis Synthesis includes both Correlated Active Problems AND Davis CoPilot Assessment
- [ ] Recommended Actions table + Decision guidance bullets BOTH present
- [ ] Appendix A present with the SLI expression verbatim
- [ ] No DQL anywhere except Appendix A query log
- [ ] Footer is `**Links:**` + `*End of Briefing*` (no tool-version signature)
- [ ] Canonical deliverable is the **`.md`** — `SLO_BURN_{SLO_SLUG}_{DATE}[_SANITIZED].md`
- [ ] PDF is **only** expected if `PDF_MODE == true` — otherwise it is intentionally skipped (do not flag its absence)
- [ ] **Mermaid lint** — same rules as dt-rcf Phase 4 Mermaid lint (no colons in gantt
      task names, no quotes in sequence messages, `<br/>` for line breaks). Read dt-rcf
      Phase 4 Pre-Save Self-Check for the exact rules.

---

### Phase 5: Save Report (Markdown)

**Default: write the `.md` ONLY.** The Markdown briefing is the canonical
deliverable. Save it with the filename below.

**PDF is opt-in (`PDF_MODE == true`, set by `--pdf` / `pdf=true`) and never
fatal.** Only when `PDF_MODE`:
- Render the PDF via the inherited dt-rca path (read `~/.claude/skills/dt-rca/SKILL.md`
  Phases 3+4 for the `md-to-pdf` command, the intermediate `_pdf.md` Mermaid→ASCII
  conversion, and cleanup). NEVER use the dt-rca `Problem_{ID}_Analysis_Report`
  filename — dt-slo-burn briefings are `SLO_BURN_*`.
- **Wrap the PDF render so a missing engine is non-fatal:** if `md-to-pdf` is not
  installed (or the render errors), log exactly one line —
  `PDF engine unavailable — Markdown only` — and continue. The run still succeeds
  on the Markdown. Never hard-fail a briefing on PDF.

The PDF know-how (Mermaid→ASCII, `_pdf.md` strategy, diagram sizing) is retained
above — it is simply opt-in now, not the default.

Filenames:
```
Normal:   SLO_BURN_{SLO_SLUG}_{DATE}.md            (+ .pdf only if --pdf)
Clean:    SLO_BURN_{SLO_SLUG}_{DATE}_SANITIZED.md  (+ ...SANITIZED.pdf only if --pdf)

SLO_SLUG examples:
  SLO:"checkout-availability"    → checkout_availability
  SLO:"a3f2c8e1-1234-..."        → a3f2c8e1
```

---

### Phase 6: Output Summary

```markdown
## dt-slo-burn Briefing Generated

| Format | Filename |
|--------|----------|
| Markdown | SLO_BURN_{SLUG}.md |
| PDF | SLO_BURN_{SLUG}.pdf (or "skipped (MD-only default; pass --pdf to enable)") |
| Log | SLO_BURN_{SLUG}.log |

**SLO:** {SLO_NAME} ({SLO_ID})
**Horizon:** {BURN_HORIZON}
**Verdict:** {VERDICT}
**Burn Rates:** long {long_burn_rate}× / short {short_burn_rate}× (threshold {THRESHOLD}×)
**Workers Dispatched:** W-burn, W-contributors, W-davis{+ W-forecast}

### Headline
- {one-sentence VERDICT reason}
- Top failing endpoint: {endpoint} ({status}, {count} fails)
- Active Davis problems: {count}
[+ Forecast exhaustion datetime if FORECAST_MODE]
```

If CLEAN_MODE, append the sanitization key to console (never to file) per dt-rca rules.

---

## ERROR HANDLING

### SLO not found
```markdown
## Error: SLO Not Found

Anchor `SLO:"{SLO_ANCHOR}"` could not be resolved in this tenant.

**Possible causes:**
- SLO name spelled differently — list SLOs with `dtctl get slo --plain`
- SLO opaque-ID copy/paste error
- Ambiguous fuzzy match — re-run with `SLO:"<opaque-id>"` from the candidate list
- SLO exists in a different tenant — verify with `dtctl auth whoami`

**Try:** `dtctl get slo --plain | jq -r '.result[]? // .[] | "\(.id)  \(.name)"' | grep -i "{partial}"` to find the exact id+name pair.
```

### Template-based SLO (no Custom SLI)
```markdown
## Error: Template-Based SLO Not Supported (v1)

/dt-slo-burn v1 supports SLOs with a Custom SLI indicator only. The SLO
"{SLO_NAME}" is template-based — its `customSli.indicator` is null in
`dtctl describe slo {SLO_ID} -o json --plain`.

**Options:**
- Use the Dynatrace SLO app for template-based burn views.
- Convert this SLO to a Custom SLI (paste the equivalent indicator DQL into the
  SLO definition) and re-run /dt-slo-burn.
```

### Worker fails entirely
Orchestrator marks the corresponding section with a gap note in Appendix A and continues.
Does not auto-retry — a second worker would likely fail for the same reason. Exception:
auth failure → orchestrator runs ONE `dtctl auth refresh` and re-dispatches the failed
worker once.

### Davis CoPilot unavailable
Generate the briefing from W-burn + W-contributors + W-davis alone. Add a banner:
```
> **Note:** Davis CoPilot synthesis unavailable. Recommended Actions are derived
> from the verdict ladder and contributor concentration only.
```

### Forecast fails (insufficient history)
If `timeseries-forecast` errors with "not enough non-null values" → skip the Forecast
section, add a banner:
```
> **Forecast unavailable:** SLO history too sparse for {BURN_HORIZON} forecast horizon.
> Re-run after the SLO has accumulated more history, or use a longer HORIZON.
```

### `valid == 0` over the burn window (no traffic)
Set VERDICT to `NO TRAFFIC` (not OK). Skip burn-rate table — denominator zero is
undefined, not healthy. Banner:
```
> **No qualifying traffic in window.** SLI denominator is zero — the SLO has no
> events to evaluate. Verdict is NO TRAFFIC, not GO.
```

---

## RULES (NON-NEGOTIABLE)

### Inherited from dt-rcf

**Rules 1–37 from dt-rcf apply unchanged** — read
`~/.claude/skills/dt-rcf/SKILL.md` "RULES (NON-NEGOTIABLE)" for the full list. Key
inheritances: Smartscape-native entity model (RULE 17), sub-skills as DQL authority
(RULE 18), `not(dt.davis.is_duplicate)` required (RULE 19), parallel worker dispatch in
single message (RULE 28), absence-gate before publishing (RULES 29b, 37), DQL only in
Appendix (RULE 34), dtctl auth lazy-and-orchestrator-only (RULE 36). Login is
interactive-and-orchestrator-only; workers never log in or refresh, and the orchestrator
never races concurrent/non-interactive logins (see Auth Hardening 2026-06-03).

### New in dt-slo-burn

38. **SLO must be resolved before any other action.** Phase 0b
    `dtctl get slo --plain` (list, for name resolution) + `dtctl describe slo
    {id} -o json --plain` (detail) are the HARD GATE. **Never use
    `fetch dt.slo` — that table does not exist.** If resolution fails, exit
    with the SLO Not Found banner. Never run burn-rate math against a guessed
    SLI — the briefing's authority is the verbatim `customSli.indicator`
    extracted from the SLO record.

38a. **Custom-SLI only (v1).** If `customSli.indicator` is missing/null on the
    resolved SLO, fail fast with the Template-Based SLO banner. Do not attempt
    to synthesize an indicator from the template definition.

38b. **Never mutate the indicator DQL.** Vary the evaluation window via
    `dtctl query --default-timeframe-start/--default-timeframe-end` flags only.
    Injecting `from:` / `to:` into the indicator text breaks SLO fidelity and
    is forbidden.

39. **Multi-window burn-rate is non-negotiable.** Both LONG and SHORT windows must be
    computed and reported. A single-window burn rate is noise (short-only) or stale
    (long-only). The verdict is the AND of both crossings.

40. **The Google SRE Workbook threshold table is the truth source.** Do not invent new
    burn-rate thresholds. 14.4× for 1h fast burn, 6× for 6h, 3× for 24h, 1× for 30d.
    These map to ~5%, ~10%, ~10%, and 100% budget consumption respectively.

41. **State explicitly which window triggered.** Every verdict prose sentence must name
    the long-window rate AND the short-window rate AND identify whether both, neither,
    or one triggered. "FAST BURN" with no rate numbers is invalid.

42. **Forecast is opt-in and confidence-gated.** Only run W-forecast when `--forecast`
    is passed. If `SLO_HISTORY_DAYS < 7`, emit the forecast warning banner in Phase 0b
    but continue — the user opted in, deliver the forecast with a wide-band caveat.

43. **NO TRAFFIC ≠ GO.** An SLI indicator that returns an empty `sli` pool over
    both windows is not healthy — it has no signal at all. Verdict is
    `NO TRAFFIC`. Never report `GO` when the indicator produced zero
    non-null values.

44. **Filename = `SLO_BURN_{SLO_SLUG}_{DATE}[_SANITIZED]`** — never the dt-rca or dt-rcf
    naming. SLO_SLUG is the SLO name (slugified) or first 8 chars of UUID.

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section "PDF DIAGRAM RULES".
Summary: no Mermaid in PDFs — ASCII art only, inside plain code blocks, 80-char max width.

## MERMAID SYNTAX RULES

Identical to dt-rcf. Read `~/.claude/skills/dt-rcf/SKILL.md` section "MERMAID SYNTAX
RULES" — same gantt/sequence/label sanitization rules apply unchanged.

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section "DIAGRAM SIZING RULES".

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously until the briefing
is saved.

**Do not stop for confirmation between phases. Do not ask questions. Generate the complete
briefing.**
