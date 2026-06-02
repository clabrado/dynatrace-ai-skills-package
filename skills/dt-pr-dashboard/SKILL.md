---
name: dt-pr-dashboard
description: >
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
operational view of the incident.

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
| `--epoch true` | **default** | Pin every tile and the dashboard defaultTimeframe to the absolute incident window. |
| `--epoch false` | optional | Use relative `now()-Xh`. No tile-level timeframe overrides. |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- dynatrace-for-ai skills installed: `dt-obs-problems`, `dt-dql-essentials`, `dt-obs-logs`, `dt-obs-tracing`, `dt-obs-services`, `dt-app-dashboards`

## EXECUTION PROTOCOL

### Phase 0 — Parse & Validate

Extract `PROBLEM_ID` matching `P-\d+`. Extract optional `--title` value.

### Phase 1 — Problem Triage

**Step 1.1 — Fetch problem record:**
```dql
fetch dt.davis.problems, from:now()-24h
| filter display_id == "<PROBLEM_ID>"
| fields
    display_id, event.name, event.category, event.status,
    event.start, event.end,
    root_cause_entity_id, root_cause_entity_name,
    affected_entity_ids, affected_entity_names, affected_entity_types,
    dt.davis.affected_users_count, dt.davis.event_ids,
    event.description
```

Extract: `PROBLEM_START`, `PROBLEM_END`, `PROBLEM_CATEGORY`, `ROOT_ENTITY_ID`, `ROOT_ENTITY_NAME`, `AFFECTED_IDS[]`, `USER_IMPACT`

Compute timeframe:
- `WINDOW_START` = `event.start - 30min` as epoch ms AND as quoted ISO 8601
- `WINDOW_END` = `event.end + 30min` (or current epoch ms if ACTIVE)

**Step 1.2 — Deployment trigger check** (parallel with 1.3):
```dql
fetch events, from:now()-6h
| filter event.type == "CUSTOM_DEPLOYMENT"
| dedup event.id
| fields timestamp, event.name, event.description, affected_entity_ids
| sort timestamp desc | limit 20
```

**Step 1.3 — Related ACTIVE problems** (parallel with 1.2):
```dql
fetch dt.davis.problems, from:now()-24h
| filter not(dt.davis.is_duplicate) and event.status == "ACTIVE"
| filter matchesPhrase(arrayToString(affected_entity_names, delimiter:","), "<ROOT_ENTITY_NAME_FRAGMENT>")
| fields display_id, event.name, event.category, event.start,
         root_cause_entity_name, affected_entity_names, dt.davis.affected_users_count
| sort event.start desc | limit 10
```

### Phase 2 — Category-Adaptive Signal Queries

**Dashboard timeframe rules (--epoch true, default):**
Three layers must ALL be set:
1. `defaultTimeframe` → epoch ms strings
2. Each data tile → `tileTimeframe` with `tileTimeframeEnabled: true`
3. Each tile DQL → `from:"<WINDOW_START_ISO>", to:"<WINDOW_END_ISO>"` (both required)

**Named exceptions (always use relative time):**
- Davis problems queries: `from:now()-24h`
- Verification tiles: `from:now()-15m` or `from:now()-30m`

#### ERROR category signal queries

**Q-ERR-1: Error rate % by service (lineChart)**
```dql
timeseries total = sum(dt.service.request.count, default:0), nonempty:true, by:{dt.entity.service}
| lookup [timeseries errors = sum(dt.service.request.failure_count, default:0), nonempty:true, by:{dt.entity.service}],
  sourceField:dt.entity.service, lookupField:dt.entity.service, prefix:"err."
| fieldsAdd failureRate = err.errors[] / total[] * 100
| fieldsAdd service.name = entityName(dt.entity.service)
| filter matchesValue(toString(dt.entity.service), "SERVICE-*")
| fields timeframe, interval, service.name, dt.entity.service, failureRate
| sort arrayAvg(failureRate) desc | limit 10
```

**Q-ERR-2: HTTP 5xx count by service (lineChart)**, **Q-ERR-3: HTTP status distribution (barChart)**, **Q-ERR-4: Error log volume by workload (areaChart)**, **Q-ERR-5: Top error log messages (table)**, **Q-ERR-6: Affected service health (honeycomb)** — see full SKILL.md for complete query bodies.

#### SLOWDOWN category: Q-SLW-1/2/3 (p50/p90/p99 latency lineCharts) + Q-SLW-4 (slow endpoints table)
#### AVAILABILITY category: Q-AVL-1/2/3 (honeycomb + singleValue + events table)
#### RESOURCE_CONTENTION: Q-RES-1/2/3/4 (CPU/memory lineCharts)

### Phase 3 — Layout Planning

Determine:
- `AFFECTED_ENTITY_COUNT` ≥ 3 → honeycomb at y:0; 1-2 → singleValue KPI tiles
- `HAS_K8S` → include infrastructure section
- `DASHBOARD_TITLE` from `--title` or auto-generated

### Phase 4 — Dashboard Construction

Read `~/.claude/skills/dt-app-dashboards/SKILL.md` for JSON structure and deploy workflow.

**Validate ALL queries before embedding:**
```bash
dtctl query '<DQL>' --plain
```

**Standard layout template (24-column grid):**
```
y:0   h:4   Honeycomb (w:9) | Problems table (w:15)
y:4   h:1   DIVIDER: "AFFECTED SERVICES"
y:5   h:3   Service details table (w:24)
y:8   h:1   DIVIDER: category-label
y:9   h:4   4 category-adaptive tiles (each w:6)
y:13  h:1   DIVIDER: "TIMELINE"
y:14  h:4   2 timeseries tiles (each w:12)
y:18  h:1   DIVIDER: "BLAST RADIUS"
y:19  h:4   Active problems table (w:24)
y:23  h:1   DIVIDER: "INFRASTRUCTURE" (skip if no K8s)
y:24  h:4   4 infra tiles (each w:6)
y:28  h:1   DIVIDER: "VERIFICATION"
y:29  h:3   Verification singleValue (w:8) | Verification timeseries (w:16)
```

**Deploy:**
```bash
bash ~/.claude/skills/dt-app-dashboards/scripts/deploy_dashboard.sh /tmp/<problem-id>-dashboard.json
```

## VISUALIZATION DECISION MATRIX

| Data | Viz | Rationale |
|---|---|---|
| Error rate % over time | **lineChart** | Shape/recovery of rate |
| Log error count over time by workload | **areaChart** | Volume/pressure; stacking shows which workload contributes most |
| HTTP response code distribution | **barChart** (relative) | Proportion shift: 200s → 500s |
| Latency percentiles over time | **lineChart** | Trend comparison |
| Entity binary health (N≥3) | **honeycomb** | Visual grid scan |
| Entity binary health (N=1-2) | **singleValue** per entity | Honeycomb renders poorly with <3 cells |
| Service metric matrix | **table** with coloring | Multi-dimensional |
| Problem list (text + status) | **table** | Text fields have no axis |
| CPU/memory % over time | **lineChart** | Smooth trend |
| Log message text | **table** | Text cannot be charted |
| Section header / label | **singleValue** divider | `data record(a="LABEL")` + always-blue color rule |

## QUALITY RULES

1. Never invent DQL field names.
2. Never embed an unvalidated query.
3. Timeframe completeness (--epoch true): all three layers must be set.
4. `from:` always requires `to:` in DQL.
5. Every tile ID in `tiles` must have a matching entry in `layouts`.
6. Every tile must have `davis: {enabled: false}`.
7. Visualization choice must be justified by data shape.
8. Section dividers must always use the `!= "0"` color rule trick.
9. Verification row tiles must reflect the actual root failure signature.

## SKILLS USED

| Skill | When used |
|---|---|
| `dt-obs-problems` | Phase 1 field names, query patterns |
| `dt-dql-essentials` | DQL syntax, timeseries vs makeTimeseries, entity functions |
| `dt-obs-logs` | Phase 2 log queries for ERROR/AVAILABILITY categories |
| `dt-obs-tracing` | Phase 2 span queries for SLOWDOWN category |
| `dt-obs-services` | Service metric timeseries |
| `dt-app-dashboards` | Phase 4 JSON structure, deploy_dashboard.sh |
