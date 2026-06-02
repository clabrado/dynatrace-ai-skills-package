---
name: dt-pr-notebooks
description: >
  Given a Dynatrace Problem ID, diagnose the full root cause using live telemetry
  and build a deployable Dynatrace notebook that tells the complete incident story —
  executive summary, ASCII topology diagram, step-by-step DQL queries with markdown
  annotations, and a remediation section. The notebook is ready for any SRE to use
  immediately. Composes dt-obs-problems, dt-dql-essentials, dt-obs-logs,
  dt-obs-tracing, and dt-app-notebooks. Never uses dt-rca or dt-rcf.
license: Apache-2.0
---

# dt-pr-notebooks — Problem-to-Notebook Generator

Given a Dynatrace Problem ID, this skill diagnoses the full root cause from live
telemetry, synthesizes an ASCII topology diagram, and deploys a Dynatrace notebook
that walks any SRE through the incident from trigger to fix.

## Usage

```
/dt-pr-notebooks P-260530039
/dt-pr-notebooks P-260530039 --title "EasyTrade Trade Failure RCA"
/dt-pr-notebooks P-260530039 --epoch false
/dt-pr-notebooks P-260530039 --title "EasyTrade Trade Failure RCA" --epoch false
```

## Arguments

| Argument | Default | Description |
|---|---|---|
| `P-XXXXXXX` | required | Dynatrace Problem ID |
| `--title "..."` | auto-generated | Override notebook title |
| `--epoch true` | **default** | Pin `defaultTimeframe` and every DQL section timeframe to the absolute incident window. Nothing moves when the notebook is opened later. |
| `--epoch false` | optional | Use relative `now()-Xh` throughout. All sections re-query from the current time — useful for live incidents still in progress. |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- dynatrace-for-ai skills installed: `dt-obs-problems`, `dt-dql-essentials`, `dt-obs-logs`, `dt-obs-tracing`, `dt-app-notebooks`

## EXECUTION PROTOCOL

### Phase 0 — Parse & Validate

Extract `PROBLEM_ID` matching `P-\d+`. Extract optional `--title` value.
If no valid problem ID found, print usage and stop.

### Phase 1 — Problem Triage

Read `~/.claude/skills/dt-obs-problems/SKILL.md` for field names, status values,
and query patterns. Do NOT invent field names.

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

Extract: `PROBLEM_START`, `PROBLEM_CATEGORY`, `ROOT_ENTITY_ID`, `ROOT_ENTITY_NAME`, `AFFECTED_IDS[]`, `DAVIS_EVENT_IDS[]`, `USER_IMPACT`, `DESCRIPTION`

If zero records: widen to `from:now()-7d`, retry once.

**Step 1.2 — Fetch underlying Davis events** (parallel with 1.3, 1.4):
```dql
fetch dt.davis.events, from:now()-24h
| filter in(event.id, array(<DAVIS_EVENT_IDS comma-separated quoted>))
| fields event.id, event.kind, event.type, event.name, event.description,
         event.status, event.start, affected_entity_ids
| sort event.start asc
```

**Step 1.3 — Deployment trigger check** (parallel with 1.2):
```dql
fetch events, from:now()-6h
| filter event.type == "CUSTOM_DEPLOYMENT"
| dedup event.id
| fields timestamp, event.name, event.description, affected_entity_ids
| sort timestamp desc
| limit 20
```

**Step 1.4 — Related ACTIVE problems** (parallel with 1.2, 1.3):
```dql
fetch dt.davis.problems, from:now()-24h
| filter not(dt.davis.is_duplicate) and event.status == "ACTIVE"
| filter matchesPhrase(arrayToString(affected_entity_names, delimiter:","), "<ROOT_ENTITY_NAME fragment>")
| fields display_id, event.name, event.category, event.start,
         root_cause_entity_name, affected_entity_names, dt.davis.affected_users_count
| sort event.start desc
| limit 10
```

### Phase 2 — Category-Adaptive Deep Dive

Run the queries appropriate for `PROBLEM_CATEGORY`. Read the relevant skills:
- `dt-obs-logs` for ERROR and log-based investigation
- `dt-obs-tracing` for span/trace investigation
- `dt-dql-essentials` for DQL syntax guidance

All queries MUST be validated with `dtctl query '...' --plain` before embedding.

#### ERROR category (all three in parallel)

**A) Error cascade events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "SERVICE_ERROR_RATE_INCREASED"
    or event.type == "SERVICE_CLIENT_ERROR_RATE_INCREASED"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc | limit 100
```

**B) Root cause error logs:**
```dql
fetch logs, from:now()-2h
| filter loglevel == "ERROR"
| filter dt.entity.service == "<ROOT_ENTITY_ID>"
    or matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| fields timestamp, loglevel, content, k8s.workload.name, k8s.pod.name
| sort timestamp desc | limit 50
```

**C) Error log timeseries:**
```dql
fetch logs, from:now()-3h
| filter loglevel == "ERROR"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| makeTimeseries errors = count(), interval:1m
```

#### SLOWDOWN category (all three in parallel)

**A) Slowdown cascade events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "SERVICE_SLOWDOWN" or event.type == "SERVICE_RESPONSE_TIME_DEGRADED"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc | limit 50
```

**B) Slow spans:**
```dql
fetch spans, from:now()-2h
| filter dt.entity.service == "<ROOT_ENTITY_ID>"
| summarize p99 = percentile(duration, 99), p50 = percentile(duration, 50),
            count = count(), by:{span.name}
| sort p99 desc | limit 20
```

**C) Response time timeseries:**
```dql
fetch spans, from:now()-3h
| filter dt.entity.service == "<ROOT_ENTITY_ID>"
| makeTimeseries p99_ms = percentile(duration, 99) / 1000000, interval:1m
```

### Phase 3 — Synthesis

Before building the notebook:
1. Build the causal chain (TRIGGER_EVENT, CASCADE_STEPS[], ROOT_FAILURE_MECHANISM)
2. Build ASCII topology diagram using box-and-arrow characters
3. Build incident timeline table
4. Write remediation section with post-fix verification DQL

### Phase 4 — Notebook Construction

Read `~/.claude/skills/dt-app-notebooks/SKILL.md` for JSON structure and deploy command.

**Validate ALL queries before embedding:**
```bash
dtctl query '<DQL>' --plain
```

**Compute epoch timestamps** (--epoch true, default):
- `WINDOW_START` = `event.start - 30min`
- `WINDOW_END` = `event.end + 30min` (or now() if ACTIVE)
- Convert to epoch ms for `defaultTimeframe`, ISO 8601 for DQL

**Section order:**
1. [markdown] Executive Summary + ASCII topology + incident timeline + how-to-use guide
2. [markdown] ## Step 1 — Davis Problem Record
3. [dql] Problem record query (always `from:now()-24h` — Davis requires relative bounds)
4. [markdown] ## Step 2 — Deployment / Change Trigger (skip if none found)
5. [dql] Deployment events query
6. [markdown] ## Step 3 — Error/Slowdown/Event Cascade
7. [dql] Category cascade query
8. [dql] Error/slowdown timeseries chart (areaChart)
9. [markdown] ## Step 4 — Root Cause Detail
10. [dql] Root cause logs/spans
11. [markdown] ## Step 5 — Blast Radius & User Impact
12. [dql] Active related problems (always `from:now()-24h`)
13. [markdown] ## Remediation (NO DQL code fences — remediation is prose only)
14. [dql] Post-fix verification query (always `from:now()-15m`)

**Deploy:**
```bash
bash ~/.claude/skills/dt-app-notebooks/scripts/deploy_notebook.sh /tmp/<problem-id>-notebook.json
```

## QUALITY RULES

1. Never invent DQL field names. Consult skills or run `| limit 1` to discover fields.
2. Never embed a query that hasn't been validated.
3. ASCII diagram must use actual entity names and IDs from the data.
4. ROOT_FAILURE_MECHANISM must be a concrete technical statement.
5. Markdown annotations must be written for an SRE encountering this at 3am.
6. Epoch time is the default (`--epoch true`).
7. Named exceptions always use relative time: Davis problem queries (`from:now()-24h`), verification query (`from:now()-15m`).
8. The verification DQL must return 0 records when the fix works.

## SKILLS USED

| Skill | When used |
|---|---|
| `dt-obs-problems` | Phase 1 field names, query patterns, problem schema |
| `dt-dql-essentials` | DQL syntax, field escaping, makeTimeseries vs timeseries |
| `dt-obs-logs` | Phase 2 log queries for ERROR/AVAILABILITY categories |
| `dt-obs-tracing` | Phase 2 span queries for SLOWDOWN category |
| `dt-app-notebooks` | Phase 4 JSON structure, deploy_notebook.sh, section types |
