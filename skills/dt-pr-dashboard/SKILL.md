---
name: dt-pr-dashboard
description: >-
  Given a Dynatrace Problem ID, diagnose the full root cause using live telemetry
  and build a deployable Dynatrace dashboard that turns the incident into an
  operational view — entity health honeycomb, problem summary, category-adaptive
  signal tiles, affected service metrics table, blast radius, and a verification
  row. Applies expert visualization selection logic based on the problem category
  and metric type. Style: Services Overview dashboard (clean section dividers,
  coloring rules, deep-links). Composes dt-obs-problems, dt-dql-essentials,
  dt-obs-logs, dt-obs-tracing, dt-obs-services, and dt-app-dashboards. Never
  uses dt-rca or dt-rcf.
license: Apache-2.0
---

# dt-pr-dashboard — Problem-to-Dashboard Generator

Given a Dynatrace Problem ID, this skill diagnoses the full root cause from live
telemetry and builds a Dynatrace dashboard that gives any operator an immediate
operational view of the incident — what's broken, how much, which services are
affected, and whether the fix has taken hold.

Dashboards are NOT notebooks. They are not narrative — they are simultaneous. Every
tile must be independently interpretable at a glance. Visualization choices are as
important as the queries themselves.

## Usage

```
/dt-pr-dashboard P-260530039
/dt-pr-dashboard P-260530039 --title "EasyTrade Incident — Ops View"
/dt-pr-dashboard P-260530039 --epoch false
/dt-pr-dashboard P-260530039 --title "EasyTrade Incident — Ops View" --epoch false
```

## Arguments

| Argument | Default | Description |
|---|---|---|
| `P-XXXXXXX` | required | Dynatrace Problem ID |
| `--title "..."` | auto-generated | Override dashboard title |
| `--epoch true` | **default** | Pin every tile and the dashboard defaultTimeframe to the absolute incident window. Nothing moves when the dashboard is opened later. |
| `--epoch false` | optional | Use relative `now()-Xh` covering the incident window. No tile-level timeframe overrides — global time picker controls all tiles so operators can shift the window live. |

---

## EXECUTION PROTOCOL

### Phase 0 — Parse & Validate

Extract `PROBLEM_ID` matching `P-\d+`. Extract optional `--title` value.
If no valid problem ID found, print usage and stop.

---

### Phase 1 — Problem Triage

Read `~/.claude/skills/dt-obs-problems/SKILL.md` for field names, status values,
and query patterns. Do NOT invent field names.

**Step 1.1 — Fetch problem record:**

```dql
fetch dt.davis.problems, from:now()-7d
| filter display_id == "<PROBLEM_ID>"
| fields
    display_id, event.name, event.category, event.status,
    event.start, event.end,
    root_cause_entity_id, root_cause_entity_name,
    affected_entity_ids, affected_entity_names, affected_entity_types,
    dt.davis.affected_users_count, dt.davis.event_ids,
    event.description
```

Extract:
- `PROBLEM_START` — `event.start`
- `PROBLEM_END` — `event.end` (null if ACTIVE)
- `PROBLEM_CATEGORY` — `event.category`
- `ROOT_ENTITY_ID` — `root_cause_entity_id`
- `ROOT_ENTITY_NAME` — `root_cause_entity_name`
- `AFFECTED_IDS[]` — `affected_entity_ids`
- `AFFECTED_NAMES[]` — `affected_entity_names`
- `USER_IMPACT` — `dt.davis.affected_users_count`
- `WINDOW_START` — `PROBLEM_START - 30min` as **epoch milliseconds** (e.g. `1779816900000`) for `defaultTimeframe`; as **quoted ISO 8601** (e.g. `"2026-05-26T17:35:00Z"`) for DQL `from:` clauses
- `WINDOW_END` — `PROBLEM_END + 30min`, or current epoch ms if still ACTIVE

If zero records: widen to `from:now()-30d`, retry once. Still nothing → report and stop.
Never use a 24h lookback for Davis problems — a problem that started more than 24h ago is silently missed.

**Step 1.2 — Deployment trigger check** (parallel with 1.3 on dtctl ≥ v0.28.0; run serially on older dtctl — see 4.1):

```dql
fetch events, from:now()-6h
| filter event.type == "CUSTOM_DEPLOYMENT"
| dedup event.id
| fields timestamp, event.name, event.description, affected_entity_ids
| sort timestamp desc
| limit 20
```

Set `IS_DEPLOYMENT_TRIGGERED = true` if a deployment exists within 30 min before
`PROBLEM_START`.

**Step 1.3 — Related ACTIVE problems** (parallel with 1.2 on dtctl ≥ v0.28.0; serially on older dtctl):

```dql
fetch dt.davis.problems, from:now()-7d
| filter not(dt.davis.is_duplicate) and event.status == "ACTIVE"
| filter matchesPhrase(arrayToString(affected_entity_names, delimiter:","), "<ROOT_ENTITY_NAME_FRAGMENT>")
| fields display_id, event.name, event.category, event.start,
         root_cause_entity_name, affected_entity_names, dt.davis.affected_users_count
| sort event.start desc
| limit 10
```

---

### Phase 2 — Category-Adaptive Signal Queries

**Dashboard timeframe rules — `--epoch true` (default):**

Compute once from the problem record:
- `WINDOW_START` = `event.start - 30min`
- `WINDOW_END` = `event.end + 30min` (or current epoch ms if ACTIVE)
- `WINDOW_START_MS` / `WINDOW_END_MS` = epoch milliseconds (e.g. `1779816900000`)
- `WINDOW_START_ISO` / `WINDOW_END_ISO` = quoted ISO 8601 (e.g. `"2026-05-26T17:35:00Z"`)

Three layers must ALL be set:
1. **`defaultTimeframe`** → `{"from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>"}` as **epoch ms strings**. The dashboard API silently ignores ISO 8601 here and falls back to `now()-3h`. Convert with: `python3 -c "from datetime import datetime,timezone as tz; print(int(datetime(2026,5,26,17,35,tzinfo=tz.utc).timestamp()*1000))"`
2. **Each data tile** → `"timeframe": {"tileTimeframe": {"from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>"}, "tileTimeframeEnabled": true}` — without this, the global UI time picker overrides the tile even if `defaultTimeframe` is set
3. **Each tile DQL query** → `from:"<WINDOW_START_ISO>", to:"<WINDOW_END_ISO>"` — **both `from:` and `to:` required**. `from:` without `to:` silently defaults to `now()`, scanning the incident window to the present (wrong for closed incidents). For `makeTimeseries from:/to:` parameters, wrap the strings: `from:toTimestamp("<ISO>"), to:toTimestamp("<ISO>")`.

**A tile window narrower than the dashboard window needs BOTH layers 2 and 3** — set `tileTimeframe` AND the query's `from:`/`to:`. The chart axis follows the query window, so a tileTimeframe alone does not narrow the chart.

**Named exceptions — tiles that ALWAYS use relative time (even with `--epoch true`):**

These tiles intentionally break the epoch pattern. The deploy validator will warn on them;
that warning is expected and not blocking. Set the relative window in BOTH the query and the
tile's `tileTimeframe` (see the narrower-window rule above).

| Tile(s) | Time filter | Reason |
|---|---|---|
| `t-problems`, `t-blast` (Davis problems queries) | `from:now()-7d` | Davis requires relative bounds; absolute timestamps on `dt.davis.problems` are unreliable |
| Any other Davis tile (KPI user count, crash rate from Davis) | `from:now()-7d` | Same — Davis data source constraint |
| `t-verify-kpi`, `t-verify-ts` (verification row) | `from:now()-15m` or `from:now()-30m` | Live state — must show whether the fix is currently holding, not whether it was fixed at incident time |

**Dashboard timeframe rules — `--epoch false`:**
- Compute `HOURS_AGO = ceil((now_ms - WINDOW_START_MS) / 3600000) + 1`
- `defaultTimeframe` → `{"from": "now()-<HOURS_AGO>h", "to": "now()"}`
- **No `tileTimeframeEnabled`** on any tile — global time picker controls all
- **No `from:`/`to:` in DQL queries** — let the UI time picker pass the window; exception: Davis problems tiles always need explicit `from:` (Davis requires time bounds regardless)
- Use `timeseries` (not `makeTimeseries`) for metric time series.
- Use `makeTimeseries` only for log/span aggregations where `timeseries` doesn't apply.
- Always resolve entity names inline with `entityName()` or `fieldsAdd ... = entityName(...)`.
- Use `timeseries` (not `makeTimeseries`) for metric time series.
- Use `makeTimeseries` only for log/span aggregations where `timeseries` doesn't apply.
- Always resolve entity names inline with `entityName()` or `fieldsAdd ... = entityName(...)`.

Read relevant skills before writing queries:
- `dt-obs-logs` for log-based queries
- `dt-obs-tracing` for span queries
- `dt-obs-services` for service metric queries
- `dt-dql-essentials` for DQL syntax

#### ERROR category — validate these 6 queries:

**Q-ERR-1: Error rate % by service (lineChart)**
```dql
timeseries total = sum(dt.service.request.count, default:0), nonempty:true,
           by:{dt.entity.service}
| lookup [timeseries errors = sum(dt.service.request.failure_count, default:0),
           nonempty:true, by:{dt.entity.service}],
  sourceField:dt.entity.service, lookupField:dt.entity.service, prefix:"err."
| fieldsAdd failureRate = err.errors[] / total[] * 100
| fieldsAdd service.name = entityName(dt.entity.service)
| filter matchesValue(toString(dt.entity.service), "SERVICE-*")
| fields timeframe, interval, service.name, dt.entity.service, failureRate
| sort arrayAvg(failureRate) desc
| limit 10
```
*Scope to affected services: add `| filter in(dt.entity.service, array("<ID1>","<ID2>"))` after the filter.*
→ Viz: **lineChart** — trend shape reveals when error rate spiked and whether it's recovering.

**Q-ERR-2: HTTP 5xx count by service (lineChart)**
```dql
timeseries errors = sum(dt.service.request.count, default:0), nonempty:true,
           by:{dt.entity.service},
           filter: http.response.status_code >= 500 and http.response.status_code <= 599
| fieldsAdd service.name = entityName(dt.entity.service)
| fields timeframe, interval, service.name, errors
| sort arraySum(errors) desc
| limit 10
```
→ Viz: **lineChart** — count of HTTP 5xx over time; reveals blast radius by service.

**Q-ERR-3: HTTP status distribution (barChart)**
```dql
timeseries requests = sum(dt.service.request.count),
           by:{http.response.status_code}
| fields timeframe, interval, http.response.status_code, requests
| sort http.response.status_code asc
| limit 10
```
→ Viz: **barChart** with `valueRepresentation: "relative"` — shows the shift from healthy
(green 200s dominate) to broken (red 500s dominate). Time-series chart cannot show this
proportional shift as clearly.

**Q-ERR-4: Error log volume by workload (areaChart)**
```dql
fetch logs
| filter loglevel == "ERROR"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| makeTimeseries errors = count(), interval:1m, by:{k8s.workload.name}
```
*Derive workload fragment from ROOT_ENTITY_NAME (e.g., `[eks-live][easytrade] BrokerService` → `broker`).*
→ Viz: **areaChart** — volume of error log lines over time, stacked by workload.
areaChart chosen (not lineChart) because log volume is cumulative pressure, not a rate.

**Q-ERR-5: Top error log messages (table)**
```dql
fetch logs
| filter loglevel == "ERROR"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| summarize count = count(), example = takeFirst(content), by:{content}
| sort count desc
| limit 20
```
→ Viz: **table** — distinct error messages with frequency. Shows the actual exception text.
Never use a chart for log message text — no meaningful axis exists.

**Q-ERR-6: Affected service health (honeycomb)**
```dql
fetch dt.entity.service
| filter in(id, array("<AFFECTED_ID_1>","<AFFECTED_ID_2>"))
| lookup [
    fetch dt.davis.problems, from:now()-7d
    | filter event.status == "ACTIVE"
    | expand affected_entity_ids
  ], sourceField:id, lookupField:affected_entity_ids
| fieldsAdd affected = if(isNotNull(lookup.affected_entity_ids), "Problem", else:"Healthy")
| fields affected, id, entity.name
```
→ Viz: **honeycomb** — binary HEALTHY/PROBLEM color grid. Use when N≥3 affected entities.
For N=1–2 entities, skip honeycomb and use singleValue KPI tiles instead.

#### SLOWDOWN category — validate these 4 queries:

**Q-SLW-1: p50 latency by service (lineChart)**
```dql
timeseries latency_p50 = percentile(dt.service.request.response_time, 50),
           by:{dt.entity.service}
| fieldsAdd service.name = entityName(dt.entity.service)
| fields timeframe, interval, service.name, latency_p50
| sort arrayAvg(latency_p50) desc | limit 10
```
→ Viz: **lineChart**, units: microsecond→ms.

**Q-SLW-2: p90 latency by service (lineChart)**
Same as Q-SLW-1 but `percentile(..., 90)`.
→ Viz: **lineChart**

**Q-SLW-3: p99 latency by service (lineChart)**
Same as Q-SLW-1 but `percentile(..., 99)`.
→ Viz: **lineChart** — p99 is the first to show degradation; include alongside p50 for comparison.

**Q-SLW-4: Slow endpoints (table)**
```dql
fetch spans
| filter dt.entity.service == "<ROOT_ENTITY_ID>"
| filter isNotNull(endpoint.name)
| summarize p99_ms = percentile(duration, 99)/1000000,
            p50_ms = percentile(duration, 50)/1000000,
            count = count(),
            by:{endpoint.name}
| sort p99_ms desc | limit 20
```
→ Viz: **table** with `unitsOverrides` for ms columns.
Endpoint-level p99 shows WHICH operation is slow — cannot visualize on a chart effectively.

#### AVAILABILITY category — validate these 3 queries:

**Q-AVL-1: Entity health honeycomb** (see Q-ERR-6 pattern)
→ Viz: **honeycomb**

**Q-AVL-2: Error log count singleValue**
```dql
fetch logs
| filter loglevel == "ERROR" or loglevel == "WARN"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| summarize count = count()
```
→ Viz: **singleValue** with red color threshold (count > 0 = red).

**Q-AVL-3: Availability events (table)**
```dql
fetch events
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "AVAILABILITY_EVENT"
    or event.type == "SERVICE_UNEXPECTED_UNAVAILABILITY"
| dedup event.id
| fields timestamp, event.type, event.name, event.description, event.status
| sort timestamp desc | limit 30
```
→ Viz: **table** — event text has no meaningful chart axis.

#### RESOURCE_CONTENTION category — validate these 4 queries:

**Q-RES-1: Process CPU % (lineChart)**
```dql
timeseries cpu = avg(dt.process.cpu.usage), by:{dt.entity.process_group_instance, dt.entity.host}
| fieldsAdd pgi = entityName(dt.entity.process_group_instance)
| fieldsAdd host.name = entityName(dt.entity.host)
| fields timeframe, interval, pgi, host.name, cpu
| sort arrayAvg(cpu) desc | limit 10
```
→ Viz: **lineChart**

**Q-RES-2: Process memory % (lineChart)**
Same as Q-RES-1 but `dt.process.memory.usage`.
→ Viz: **lineChart**

**Q-RES-3: K8s workload CPU (lineChart)**
```dql
timeseries cpu = avg(dt.kubernetes.container.cpu_usage),
           by:{k8s.namespace.name, dt.entity.cloud_application}
| fieldsAdd workload = entityName(dt.entity.cloud_application)
| fields timeframe, interval, workload, cpu, k8s.namespace.name
| sort arrayAvg(cpu) desc | limit 10
```
→ Viz: **lineChart**

**Q-RES-4: K8s workload memory (lineChart)**
Same as Q-RES-3 but `dt.kubernetes.container.memory_working_set`.
→ Viz: **lineChart**

#### CUSTOM_ALERT — treat as ERROR with business event logging:

Use Q-ERR-4 and Q-ERR-5 log queries, plus add a biz events tile:
```dql
fetch bizevents
| filter matchesPhrase(event.type, "<alert-fragment>")
| fields timestamp, event.type, event.provider, content
| sort timestamp desc | limit 30
```
→ Viz: **table**

---

### Phase 3 — Layout Planning

Before constructing JSON, determine:

1. **AFFECTED_ENTITY_COUNT** — count of `AFFECTED_IDS[]`
   - ≥ 3 entities → use honeycomb at y:0
   - 1–2 entities → replace honeycomb with 2 singleValue KPI tiles (user impact, error count)

2. **HAS_K8S** — check if any `affected_entity_types` contains `CLOUD_APPLICATION` or
   `KUBERNETES_CLUSTER` → include K8s metrics section if true

3. **ADAPTIVE_TILE_COUNT** — count tiles needed for PROBLEM_CATEGORY:
   - ERROR: 4 tiles (error rate, 5xx, status dist, log volume)
   - SLOWDOWN: 4 tiles (p50, p90, p99, slow endpoints table)
   - AVAILABILITY: 3 tiles (honeycomb, error count singleValue, events table)
   - RESOURCE_CONTENTION: 4 tiles (proc CPU, proc memory, k8s CPU, k8s memory)

4. **DASHBOARD_TITLE** — `--title` arg or default:
   `"<PROBLEM_ID> — <event.name> (<ROOT_ENTITY_NAME>) Ops View"`

---

### Phase 4 — Dashboard Construction

Read `~/.claude/skills/dt-app-dashboards/SKILL.md` and
`~/.claude/skills/dt-app-dashboards/references/create-update.md`
for the JSON structure, validation, and mandatory deploy workflow.

**4.1 — Validate ALL queries before embedding:**

```bash
dtctl query '<DQL>' --plain
```

Check `dtctl version` first. On dtctl **< v0.28.0**, run validation queries **serially** (one
`dtctl` process at a time): concurrent invocations race on OAuth refresh-token rotation and fail
with `invalid_grant` (dtctl issue #248, fixed by PR #249 in v0.28.0).

For validation, run all queries with `from:"<WINDOW_START_ISO>", to:"<WINDOW_END_ISO>"` —
the same absolute window used in the final JSON. Always include both `from:` and `to:`.
`timeseries` accepts the same quoted ISO 8601 syntax as `fetch`.

**4.2 — Dashboard JSON structure:**

```json
{
  "name": "<DASHBOARD_TITLE>",
  "type": "dashboard",
  "content": {
    "version": 21,
    "variables": [],
    "annotations": [],
    "importedWithCode": false,
    "settings": {
      "defaultTimeframe": { "enabled": true, "value": { "from": "<WINDOW_START_EPOCH_MS>", "to": "<WINDOW_END_EPOCH_MS>" } },
      "gridLayout": { "mode": "responsive" }
    },
    "tiles": { ... },
    "layouts": { ... }
  }
}
```

**4.3 — Standard layout template:**

Use string tile IDs. The `layouts` key MUST match the `tiles` key exactly.
Quote `"y"` in YAML (bare `y` parses as boolean). In JSON, no quoting needed.

```
y:0   h:4   Honeycomb (w:9, id:"t-honeycomb") | Problems table (w:15, id:"t-problems")
y:4   h:1   DIVIDER: "AFFECTED SERVICES" (w:24, id:"div-services")
y:5   h:3   Service details table (w:24, id:"t-services")
y:8   h:1   DIVIDER: category-label (w:24, id:"div-signals")
y:9   h:4   4 category-adaptive tiles (each w:6, ids:"t-sig-a","t-sig-b","t-sig-c","t-sig-d")
y:13  h:1   DIVIDER: "TIMELINE" (w:24, id:"div-timeline")
y:14  h:4   2 timeseries tiles (each w:12, ids:"t-ts-a","t-ts-b")
y:18  h:1   DIVIDER: "BLAST RADIUS" (w:24, id:"div-blast")
y:19  h:4   Active problems table (w:24, id:"t-blast")
y:23  h:1   DIVIDER: "INFRASTRUCTURE" (w:24, id:"div-infra")  [skip if no K8s]
y:24  h:4   4 infra tiles (each w:6, ids:"t-inf-a","t-inf-b","t-inf-c","t-inf-d")
y:28  h:1   DIVIDER: "VERIFICATION" (w:24, id:"div-verify")
y:29  h:3   Verification singleValue (w:8, id:"t-verify-kpi") | Verification timeseries (w:16, id:"t-verify-ts")
```

*If no K8s: move VERIFICATION up to y:23, remove infra section.*

**4.4 — Tile construction patterns:**

#### Section Divider Tile

Use a markdown tile. Do NOT use a `singleValue` data tile with a `data record()` query and an
always-true color rule as a divider — it renders as a blank bar.

```json
"div-services": { "type": "markdown", "content": "### AFFECTED SERVICES" }
```

#### lineChart Tile (timeseries metrics)

```json
"t-sig-a": {
  "title": "Error Rate % — Affected Services",
  "description": "Failure rate percentage over time. Healthy baseline < 1%.",
  "type": "data",
  "query": "<Q-ERR-1 DQL here>",
  "visualization": "lineChart",
  "visualizationSettings": {
    "chartSettings": {
      "truncationMode": "middle",
      "legend": { "hidden": true },
      "leftYAxisSettings": { "label": "Failure rate %" },
      "xAxisLabel": "timeframe",
      "xAxisScaling": "analyzedTimeframe",
      "fieldMapping": { "leftAxisValues": ["failureRate"], "timestamp": "timeframe" },
      "gapPolicy": "connect"
    },
    "dataMapping": { "displayedFields": ["service.name"] },
    "autoSelectVisualization": false,
    "unitsOverrides": [{
      "identifier": "failureRate", "unitCategory": "percentage", "baseUnit": "percent",
      "displayUnit": null, "decimals": 2, "suffix": "", "delimiter": false
    }]
  },
  "querySettings": { "maxResultRecords": 1000, "defaultScanLimitGbytes": 500,
                     "maxResultMegaBytes": 100, "defaultSamplingRatio": 10, "enableSampling": false },
  "davis": { "enabled": false, "davisVisualization": { "isAvailable": true } }
}
```

**fieldMapping rules:**
- `"leftAxisValues"` must match the alias in the `timeseries` or `makeTimeseries` call
- `"timestamp"` must be `"timeframe"` for both `timeseries` AND `makeTimeseries` queries. `interval` is a scalar duration string (nanoseconds), not a timestamp array — the dashboard UI cannot use it as an x-axis.
- `"displayedFields"` controls the legend label — always set to the human-readable name field

**unitsOverrides common patterns:**
- Latency (microseconds): `"unitCategory":"time","baseUnit":"microsecond"`
- CPU %: `"unitCategory":"percentage","baseUnit":"percent"`
- Memory bytes: `"unitCategory":"bytes","baseUnit":"byte"`
- Counts: `"unitCategory":"unspecified","baseUnit":"none","decimals":0`

#### areaChart Tile (log volume)

Same as lineChart but:
```json
"visualization": "areaChart",
"chartSettings": { ..., "stacked": true }
```
Use areaChart when:
- The metric is a count/volume (not a rate) — `errors = count()`
- Multiple series stacked makes visual sense (log lines per workload)
- The shape of accumulation matters more than exact value

Never use areaChart for latency percentiles or failure rate % — these are rates, not volumes.

#### barChart Tile (HTTP status distribution)

```json
"visualization": "barChart",
"chartSettings": {
  "valueRepresentation": "relative",
  "fieldMapping": { "leftAxisValues": ["requests"], "timestamp": "timeframe" }
},
"dataMapping": { "displayedFields": ["http.response.status_code"] },
"coloring": {
  "colorRules": [
    { "field": "DT.name", "comparator": "=", "value": "200", "colorMode": "custom-color", "customColor": "#0D9C29" },
    { "field": "DT.name", "comparator": "=", "value": "500", "colorMode": "custom-color",
      "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" } }
  ]
}
```
`"valueRepresentation": "relative"` — shows proportion of total, not absolute counts.
Use barChart (not lineChart) when: the distribution across categories is the point,
not the individual count trend.

#### Honeycomb Tile (entity health)

```json
"t-honeycomb": {
  "title": "Affected Entity Health",
  "type": "data",
  "query": "<Q-ERR-6 DQL>",
  "visualization": "honeycomb",
  "visualizationSettings": {
    "honeycomb": {
      "legend": { "hidden": true },
      "dataMappings": { "value": "affected" },
      "displayedFields": ["entity.name"]
    },
    "coloring": {
      "colorRules": [
        { "colorMode": "custom-color", "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" },
          "field": "affected", "value": "Problem", "comparator": "=" },
        { "colorMode": "custom-color", "customColor": "#0D9C29",
          "field": "affected", "value": "Healthy", "comparator": "=" }
      ]
    },
    "autoSelectVisualization": false
  },
  "querySettings": { "maxResultRecords": 2000, "defaultScanLimitGbytes": 500,
                     "maxResultMegaBytes": 1, "defaultSamplingRatio": 10, "enableSampling": false },
  "davis": { "enabled": false, "davisVisualization": { "isAvailable": true } },
  "timeframe": { "tileTimeframe": { "from": "now()-7d", "to": "now()" }, "tileTimeframeEnabled": true }
}
```

Note the `"timeframe"` tile-level override — honeycomb uses `dt.davis.problems` lookup
which needs a known window. Set `tileTimeframeEnabled: true` with `now()-7d` (a shorter window misses problems that started earlier).

#### Rich Table Tile (service details)

Follow the Service Details pattern from the Services Overview style guide (tile 15).
Scope it to the problem's affected service IDs by adding:
```dql
| filter in(dt.entity.service, array("<AFFECTED_ID_1>","<AFFECTED_ID_2>"))
```
after the `fieldsAdd` lines.

Coloring rules for service tables:
```json
"coloring": {
  "colorRules": [
    { "value": 0, "comparator": "≥", "field": "Failures", "colorMode": "custom-color",
      "customColor": "#0D9C29", "metadata": { "applyTo": "cell", "fields": ["Failures"] } },
    { "value": 1, "comparator": "≥", "field": "Failures", "colorMode": "custom-color",
      "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" },
      "metadata": { "applyTo": "cell", "fields": ["Failures"] } },
    { "value": 0, "comparator": "≥", "field": "FailureRate", "colorMode": "custom-color",
      "customColor": "#0D9C29", "metadata": { "applyTo": "cell", "fields": ["FailureRate"] } },
    { "value": 2, "comparator": "≥", "field": "FailureRate", "colorMode": "custom-color",
      "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" },
      "metadata": { "applyTo": "cell", "fields": ["FailureRate"] } }
  ]
}
```

#### singleValue KPI Tile (current user impact)

```json
"t-kpi-users": {
  "title": "Users Impacted",
  "type": "data",
  "query": "fetch dt.davis.problems, from:now()-7d\n| filter display_id == \"<PROBLEM_ID>\"\n| fields dt.davis.affected_users_count",
  "visualization": "singleValue",
  "visualizationSettings": {
    "singleValue": {
      "labelMode": "field",
      "label": "dt.davis.affected_users_count",
      "recordField": "dt.davis.affected_users_count",
      "colorThresholdTarget": "background",
      "trend": { "isVisible": false }
    },
    "coloring": {
      "colorRules": [
        { "value": 0, "comparator": ">", "field": "dt.davis.affected_users_count",
          "colorMode": "custom-color",
          "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" } }
      ]
    },
    "autoSelectVisualization": false
  },
  "querySettings": { "maxResultRecords": 1000, "defaultScanLimitGbytes": 500,
                     "maxResultMegaBytes": 100, "defaultSamplingRatio": 10, "enableSampling": false },
  "davis": { "enabled": false, "davisVisualization": { "isAvailable": true } }
}
```

#### Problems Table Tile (scoped to this incident)

```json
"t-problems": {
  "title": "Problems — <PROBLEM_ID>",
  "type": "data",
  "query": "fetch dt.davis.problems, from:now()-7d\n| filter not(dt.davis.is_duplicate)\n| filter matchesPhrase(arrayToString(affected_entity_names, delimiter:\",\"), \"<ROOT_ENTITY_NAME_FRAGMENT>\")\n| fields display_id, event.name, event.category, event.status, event.start,\n         root_cause_entity_name, affected_entity_names, dt.davis.affected_users_count\n| sort event.start desc\n| limit 20",
  "visualization": "table",
  "visualizationSettings": {
    "coloring": {
      "colorRules": [
        { "value": "ACTIVE", "comparator": "=", "field": "event.status", "colorMode": "custom-color",
          "customColor": { "Default": "var(--dt-colors-charts-loglevel-emergency-default, #ae132d)" },
          "metadata": { "applyTo": "cell", "fields": ["event.status"] } },
        { "value": "CLOSED", "comparator": "=", "field": "event.status", "colorMode": "custom-color",
          "customColor": "#0D9C29", "metadata": { "applyTo": "cell", "fields": ["event.status"] } }
      ]
    },
    "autoSelectVisualization": false
  },
  "querySettings": { "maxResultRecords": 1000, "defaultScanLimitGbytes": 500,
                     "maxResultMegaBytes": 100, "defaultSamplingRatio": 10, "enableSampling": false },
  "davis": { "enabled": false, "davisVisualization": { "isAvailable": true } }
}
```

**Verification row tiles (y:29):**

Two tiles:
1. `singleValue` — current error count (should be 0 post-fix). Use red threshold at > 0.
2. `lineChart` — error rate or log error volume over time. Flat/zero line = fix confirmed.

Write these using the singleValue and lineChart patterns above, scoped to the root cause
entity and filtered for the failure signature (e.g., `IDENTITY_INSERT`, OOM, timeout keyword).

**4.5 — Deploy:**

```bash
bash ~/.claude/skills/dt-app-dashboards/scripts/deploy_dashboard.sh /tmp/<problem-id>-dashboard.json
```

The script validates all queries and blocks on failures.
On success: prints URL, deletes the local file.

---

## VISUALIZATION DECISION MATRIX

This is the core expertise of this skill. For each data type, choose the visualization
that answers the right question for an operator in the context of an active incident.

| Data | Viz | Rationale |
|---|---|---|
| Error rate % over time (1–10 services) | **lineChart** | Shape/recovery of rate; each service is a series |
| Error rate % — current scalar | **singleValue** | Immediate answer: how bad right now |
| Log error count over time by workload | **areaChart** | Volume/pressure; stacked shows which workload contributes most |
| HTTP response code distribution over time | **barChart** (relative) | Proportion shift: healthy 200s → broken 500s |
| Latency percentiles (p50/p90/p99) over time | **lineChart** | Trend comparison; p99 degrades first — keep as separate series |
| Single latency value (e.g., current p99) | **singleValue** | Immediate impact: is it slow right now |
| Endpoint latency detail (N endpoints) | **table** | Too many dimensions for any chart axis |
| Entity binary health (N≥3 entities) | **honeycomb** | Visual grid scan: green=healthy, red=problem |
| Entity binary health (N=1–2) | **singleValue** per entity | Honeycomb doesn't render well for < 3 cells |
| Service metric matrix (N services × M metrics) | **table** with coloring | Multi-dimensional; cell color = threshold-based health |
| Problem list (text + status) | **table** | Text fields have no axis; status coloring drives health |
| CPU/memory % over time | **lineChart** | Smooth trend; compare hosts/PGIs |
| Log message text | **table** | Text cannot be charted; frequency column sorts by impact |
| Process count / request count | **lineChart** | Volume trend |
| Section header / label | **markdown** tile | `{"type":"markdown","content":"### LABEL"}` |
| Business event timeline | **table** | Discrete events; timestamp + type + content |
| Biz metric KPI (order count, revenue) | **singleValue** + **lineChart** | Scalar now + trend |

**Anti-patterns to avoid:**
- `lineChart` for log message text (no meaningful y-axis)
- `areaChart` for failure rate % (rate isn't a volume; misleads)
- `honeycomb` for fewer than 3 entities (renders as near-empty grid)
- `pieChart` for time-series data (pie has no time axis)
- `singleValue` for multi-entity comparisons (only shows one record)
- `table` for time-series trends (time dimension is lost in a table)
- `barChart` (non-relative) for HTTP status codes when proportion matters

---

## QUALITY RULES

1. **Never invent DQL field names.** Consult skills or run `| limit 1` to discover fields.

2. **Never embed an unvalidated query.** Run every query with `dtctl query '...' --plain`
   using `from:"<WINDOW_START_ISO>", to:"<WINDOW_END_ISO>"` during validation — the same absolute window used in the final JSON.

3. **Timeframe completeness (`--epoch true`):** All three layers must be set — `defaultTimeframe` (epoch ms), `tileTimeframeEnabled` per tile, and `from:/to:` in each DQL query. Any missing layer lets the global time picker override that tile.

4. **`from:` always requires `to:`** in DQL. Omitting `to:` silently uses `now()` as the end — for a closed incident this means scanning from the incident start to the present, returning data from two different time periods in the same query. Always specify both.

4. **Every tile ID in `tiles` must have a matching entry in `layouts`.** Missing layout
   entries cause tiles to disappear silently.

5. **Every tile must have `davis: {enabled: false}`** unless Davis AI is explicitly
   requested. Active Davis on data tiles changes visualization behavior unpredictably.

6. **Visualization choice must be justified by the data shape and the operational
   question being answered** — not by personal preference or what's easiest to query.
   Apply the decision matrix above. If unsure, default to `table` and document why
   a chart doesn't add value.

7. **Coloring rules on tables must have both a green (healthy) threshold and a red
   (critical) threshold** for any metric where "zero is good" (Failures, FailureRate,
   5xx, 4xx). A table with only red rules leaves healthy cells uncolored (grey),
   which reads as "unknown" to operators — always anchor with green at ≥ 0.

8. **Section dividers must be markdown tiles** — `{"type":"markdown","content":"### LABEL"}`.
   The `singleValue` + `data record()` divider trick renders as a blank bar.

9. **The verification row (y:29) tiles must reflect the actual root failure signature.**
   The singleValue must count the specific error (e.g., IDENTITY_INSERT log lines),
   not generic error counts. Zero = fixed.

10. **Scale tile h appropriately:**
    - Divider: `h:1`
    - KPI singleValue: `h:3`
    - Chart (lineChart/areaChart/barChart/honeycomb): `h:4`
    - Summary table: `h:3`
    - Detail table: `h:5`
    Never set `h:1` on a data chart — it renders as a collapsed strip.

---

## SKILLS USED BY THIS SKILL

| Skill | When used |
|---|---|
| `dt-obs-problems` | Phase 1 field names, query patterns, problem schema |
| `dt-dql-essentials` | DQL syntax, timeseries vs makeTimeseries, entity functions |
| `dt-obs-logs` | Phase 2 log queries for ERROR/AVAILABILITY categories |
| `dt-obs-tracing` | Phase 2 span queries for SLOWDOWN category |
| `dt-obs-services` | Service metric timeseries — `dt.service.request.*` metrics |
| `dt-app-dashboards` | Phase 4 JSON structure, deploy_dashboard.sh, tile/layout schema |

Do NOT use `dt-rca` or `dt-rcf`.

---

## EXAMPLE OUTPUT

After successful execution the user receives:

```
✓ Problem fetched: P-260530039 — Failure rate increase (ERROR, 92 users)
✓ Root cause: [eks-live][easytrade] BrokerService — SQL Error 544
✓ Category: ERROR → 6 signal queries validated
✓ Layout: 32 rows, 18 tiles (4 signal tiles + 2 timeline + infra + verification)
✓ All DQL validated: 18/18 pass
✓ Dashboard deployed: https://demo.apps.dynatrace.com/ui/apps/dynatrace.dashboards/<id>
```

## Changelog

| Date | Change |
|---|---|
| 2026-05-26 | Initial implementation — built from P-260530039 RCA session |
| 2026-05-27 | Added `--epoch` flag; three-layer timeframe rules (defaultTimeframe epoch ms, tileTimeframeEnabled per tile, from:/to: in DQL); `from:` must always pair with `to:`; `makeTimeseries` uses `timeframe` not `interval` in fieldMapping |
