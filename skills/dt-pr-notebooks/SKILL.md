---
name: dt-pr-notebooks
description: >-
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

---

## EXECUTION PROTOCOL

### Phase 0 — Parse & Validate

Extract `PROBLEM_ID` matching `P-\d+`. Extract optional `--title` value.
If no valid problem ID found, print usage and stop.

---

### Phase 1 — Problem Triage (dt-obs-problems patterns)

Read `~/.claude/skills/dt-obs-problems/SKILL.md` for field names, status values,
and query patterns. Do NOT invent field names.

**Step 1.1 — Fetch problem record** (parallel with nothing yet, run first):

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

Extract from result:
- `PROBLEM_START` — `event.start`
- `PROBLEM_CATEGORY` — `event.category` (ERROR | SLOWDOWN | AVAILABILITY | RESOURCE_CONTENTION | CUSTOM_ALERT)
- `ROOT_ENTITY_ID` — `root_cause_entity_id`
- `ROOT_ENTITY_NAME` — `root_cause_entity_name`
- `AFFECTED_IDS[]` — `affected_entity_ids`
- `AFFECTED_NAMES[]` — `affected_entity_names`
- `DAVIS_EVENT_IDS[]` — `dt.davis.event_ids`
- `USER_IMPACT` — `dt.davis.affected_users_count`
- `DESCRIPTION` — `event.description` (Davis AI narrative — use verbatim in notebook)

If zero records: widen the window to `from:now()-7d` and retry once. If still no
result, report "Problem not found" and stop.

**Step 1.2 — Fetch underlying Davis events** (run in parallel with 1.3, 1.4 once
entity IDs from 1.1 are known):

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
| filter timestamp <= toTimestamp("<PROBLEM_START>") + duration("30m")
| dedup event.id
| fields timestamp, event.name, event.description, affected_entity_ids
| sort timestamp desc
| limit 20
```

Look for deployments in the 4-hour window before `PROBLEM_START`. If found within
30 minutes of problem start → flag as `DEPLOYMENT_TRIGGER = true` and capture details.

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

---

### Phase 2 — Category-Adaptive Deep Dive

Run the queries appropriate for `PROBLEM_CATEGORY`. Read the relevant skills:
- `dt-obs-logs` for ERROR and log-based investigation
- `dt-obs-tracing` for span/trace investigation
- `dt-dql-essentials` for DQL syntax guidance

All queries MUST be validated with `dtctl query '...' --plain` before embedding
in the notebook. If a query returns 0 results, note that in the notebook markdown.

#### ERROR category

Run all three in parallel:

**A) Error cascade events** — shows how errors spread between services:
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
    or matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<AFFECTED_ID_1>")
| filter event.type == "SERVICE_ERROR_RATE_INCREASED"
    or event.type == "SERVICE_CLIENT_ERROR_RATE_INCREASED"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc
| limit 100
```

**B) Root cause error logs** — find the actual exception:
```dql
fetch logs, from:now()-2h
| filter loglevel == "ERROR"
| filter dt.entity.service == "<ROOT_ENTITY_ID>"
    or matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| fields timestamp, loglevel, content, k8s.workload.name, k8s.pod.name
| sort timestamp desc
| limit 50
```
*Note: derive workload name fragment from `ROOT_ENTITY_NAME` — strip namespace prefix,
e.g. `[eks-live][easytrade] BrokerService` → `broker`.*

**C) Error log timeseries** — volume chart for rollback verification:
```dql
fetch logs, from:now()-3h
| filter loglevel == "ERROR"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| makeTimeseries errors = count(), interval:1m
```

#### SLOWDOWN category

Run all three in parallel:

**A) Slowdown cascade events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "SERVICE_SLOWDOWN" or event.type == "SERVICE_RESPONSE_TIME_DEGRADED"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc | limit 50
```

**B) Slow spans — top operations by p99:**
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

#### AVAILABILITY category

Run all three in parallel:

**A) Availability events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "AVAILABILITY_EVENT" or event.type == "SERVICE_UNEXPECTED_UNAVAILABILITY"
    or event.type == "SYNTHETIC_NODE_OUTAGE" or event.type == "SYNTHETIC_SINGLE_WEBCHECK_OUTAGE"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc | limit 50
```

**B) Error logs around start:**
```dql
fetch logs, from:now()-2h
| filter loglevel == "ERROR" or loglevel == "WARN"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| fields timestamp, loglevel, content, k8s.workload.name, k8s.pod.name
| sort timestamp desc | limit 50
```

**C) Affected entity status:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| dedup event.id
| fields timestamp, event.type, event.name, event.description, event.status
| sort timestamp desc | limit 20
```

#### RESOURCE_CONTENTION category

Run all three in parallel:

**A) Resource contention events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| filter event.type == "RESOURCE_CONTENTION_EVENT" or event.type == "CPU_SATURATED_EVENT"
    or event.type == "MEMORY_RESOURCES_EXHAUSTED" or event.type == "LOW_DISK_SPACE"
| fields timestamp, event.type, event.status, event.description, affected_entity_ids
| sort timestamp asc | limit 50
```

**B) Host/process resource metrics** (if entity is a host or PGI):
```dql
timeseries cpu = avg(dt.host.cpu.usage), from:now()-3h,
           by:{dt.entity.host}
| filter dt.entity.host == "<HOST_ENTITY_ID>"
```

**C) Process resource events:**
```dql
fetch logs, from:now()-2h
| filter loglevel == "ERROR" or loglevel == "WARN"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| fields timestamp, loglevel, content, k8s.workload.name, k8s.pod.name
| sort timestamp desc | limit 30
```

#### CUSTOM_ALERT category

Run all two in parallel:

**A) Custom alert events:**
```dql
fetch events, from:now()-3h
| filter event.kind == "DAVIS_EVENT"
| filter matchesPhrase(arrayToString(affected_entity_ids, delimiter:","), "<ROOT_ENTITY_ID>")
| dedup event.id
| fields timestamp, event.type, event.name, event.description, event.status, affected_entity_ids
| sort timestamp desc | limit 30
```

**B) Business events or logs near alert:**
```dql
fetch logs, from:now()-2h
| filter loglevel == "ERROR" or loglevel == "WARN"
| filter matchesPhrase(k8s.workload.name, "<workload-name-fragment>")
| fields timestamp, loglevel, content, k8s.workload.name
| sort timestamp desc | limit 30
```

---

### Phase 3 — Synthesis

Before building the notebook, synthesize findings from Phases 1 and 2:

**3.1 — Build the causal chain**

Order the events chronologically and identify:
1. `TRIGGER_EVENT` — what happened first (deployment? external event? resource breach?)
2. `CASCADE_STEPS[]` — ordered list of `{ entity, what_happened, timestamp }`
3. `ROOT_FAILURE_MECHANISM` — the actual technical reason (e.g., "SQL Error 544", "OOM kill", "timeout")
4. `IS_DEPLOYMENT_CAUSED` — bool (true if deployment within 30 min of problem start)

**3.2 — Build ASCII topology diagram**

Construct a box-and-arrow diagram showing the cascade from trigger → root entity →
affected entities → user impact. Rules:
- Use `╔╗╚╝║═` for boxes
- Each box: entity name + entity ID (shortened) + key metric (error rate, latency, etc.)
- Arrows: `▼` for vertical flow, `→` for horizontal
- Trigger box at BOTTOM (labeled "DEPLOYMENT" or "TRIGGER")
- User impact box at TOP
- Root failure at the layer where the actual technical error occurs
- Keep total width ≤ 52 chars per box

**3.3 — Build incident timeline table**

```markdown
| UTC | Event |
|---|---|
| HH:MM:SS | <trigger event> |
| HH:MM:SS | <first error detection> |
| HH:MM:SS | <cascade to next entity> |
| HH:MM:SS | <Davis opens problem> |
| HH:MM:SS | <stabilization if known> |
```

**3.4 — Write remediation section**

Based on `ROOT_FAILURE_MECHANISM` and `IS_DEPLOYMENT_CAUSED`:
- If deployment-caused: include rollback steps specific to the service
- Always include: immediate P1 actions, root fix guidance, post-fix verification DQL
- Verification DQL must be a query that returns 0 records when fixed

---

### Phase 4 — Notebook Construction (dt-app-notebooks)

Read `~/.claude/skills/dt-app-notebooks/SKILL.md` for the JSON structure,
section types, deployment command, and validation rules.

**4.1 — Validate ALL queries before embedding**

For every DQL query you plan to embed in the notebook, run:
```bash
dtctl query '<DQL>' --plain
```
If a query fails: fix the syntax, retry once. If still failing: include the query
in the notebook with a markdown note that it requires time-range adjustment.
Never embed unvalidated queries.

**4.2 — Epoch computation (--epoch true, default)**

Compute once from the problem record before building JSON:

- `WINDOW_START` = `event.start - 30min`
- `WINDOW_END` = `event.end + 30min` (or current time if still ACTIVE)
- `WINDOW_START_MS` / `WINDOW_END_MS` = epoch milliseconds as strings (e.g. `"1779816900000"`)
- `WINDOW_START_ISO` / `WINDOW_END_ISO` = quoted ISO 8601 for DQL (e.g. `"2026-05-26T17:35:00Z"`)

Convert timestamps with:
```bash
python3 -c "from datetime import datetime,timezone as tz; print(int(datetime(YYYY,MM,DD,HH,MM,tzinfo=tz.utc).timestamp()*1000))"
```

**--epoch false:** Skip epoch computation. Use `"from": "now()-Xh"` for `defaultTimeframe`
where X covers the incident window, and use `from:now()-Xh` in all DQL queries.

**Exceptions that always use relative time regardless of --epoch flag:**

| Section | Time filter | Reason |
|---|---|---|
| Davis problem record (Step 1.1) | `from:now()-24h` (or `-7d`) | Davis requires relative bounds; absolute bounds on `dt.davis.problems` are unreliable |
| Related active problems (Step 1.4) | `from:now()-24h` | Live state — must show currently active, not historic |
| Verification query (Section 14) | `from:now()-15m` | Live state — verifies the fix is still holding right now |

**Notebook JSON structure**

Build the notebook as a JSON file at `/tmp/<problem-id>-notebook.json`.
Follow the schema from `dt-app-notebooks` exactly:

```json
{
  "name": "<NOTEBOOK_TITLE>",
  "type": "notebook",
  "content": {
    "version": "7",
    "defaultTimeframe": { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" },
    "sections": [ ... ]
  }
}
```

**--epoch false:** set `"defaultTimeframe": { "from": "now()-Xh", "to": "now()" }`

**Default title:** `<PROBLEM_ID> RCA — <event.name> (<ROOT_ENTITY_NAME>)`
**Title override:** use `--title` argument if provided.

**4.3 — Section order**

Build sections in this order:

```
1.  [markdown]  Executive Summary
                  - 2-3 sentence summary (what, when, who was impacted)
                  - ASCII topology diagram (from Phase 3.2)
                  - Incident timeline table (from Phase 3.3)
                  - How to use this notebook (step guide)

2.  [markdown]  ## Step 1 — Davis Problem Record
                  - What this query shows, what to look for

3.  [dql]       Problem record query (Step 1.1 DQL)
                  title: "Davis Problem Record — <PROBLEM_ID>"
                  ⚠ EXCEPTION: always from:now()-24h (Davis requires relative bounds)
                  timeframe: omit tileTimeframe; use from:now()-24h in DQL

4.  [markdown]  ## Step 2 — Deployment / Change Trigger  (SKIP if no deployments found)
                  - Explain deployment correlation, time delta to first error

5.  [dql]       Deployment events query (Step 1.3 DQL)   (SKIP if no deployments found)
                  title: "Deployment Events Near Problem Start"
                  timeframe: { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" }
                  DQL: from:toTimestamp("<WINDOW_START_ISO>"), to:toTimestamp("<WINDOW_END_ISO>")

6.  [markdown]  ## Step 3 — Error/Slowdown/Event Cascade
                  - Explain what the cascade query shows, how to read it

7.  [dql]       Category cascade query (Phase 2 Query A)
                  title: "Cascade Events — <ROOT_ENTITY_NAME>"
                  timeframe: { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" }
                  DQL: from:toTimestamp("<WINDOW_START_ISO>"), to:toTimestamp("<WINDOW_END_ISO>")

8.  [dql]       Error/slowdown timeseries chart (Phase 2 Query C)
                  title: "<metric> Over Time — Use for Rollback Verification"
                  visualization: "areaChart", autoSelectVisualization: false
                  timeframe: { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" }
                  DQL: from:toTimestamp("<WINDOW_START_ISO>"), to:toTimestamp("<WINDOW_END_ISO>")

9.  [markdown]  ## Step 4 — Root Cause Detail
                  - Full explanation of ROOT_FAILURE_MECHANISM
                  - Quote the key exception/metric/event verbatim
                  - Cite which service/layer owns it

10. [dql]       Root cause deep query (Phase 2 Query B — logs/spans/metrics)
                  title: "Root Cause Logs / Spans — <ROOT_ENTITY_NAME>"
                  timeframe: { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" }
                  DQL: from:toTimestamp("<WINDOW_START_ISO>"), to:toTimestamp("<WINDOW_END_ISO>")

11. [markdown]  ## Step 5 — Blast Radius & User Impact
                  - User count, related active problems

12. [dql]       Active related problems query (Step 1.4 DQL)
                  title: "Active Problems — Blast Radius"
                  ⚠ EXCEPTION: always from:now()-24h (live state — must reflect current problems)
                  timeframe: omit tileTimeframe; use from:now()-24h in DQL

13. [markdown]  ## Remediation
                  - P1/P2 action table
                  - Root fix guidance
                  - One sentence: "Run the verification query below — zero records confirms fix applied."
                  - NO DQL code blocks in this markdown section

14. [dql]       Post-fix verification query
                  title: "Verification — Zero <ROOT_FAILURE_MECHANISM fragment> Confirms Fix Applied"
                  query: a targeted log/event fetch that returns 0 records once the fix is applied
                  ⚠ EXCEPTION: always from:now()-15m (live state — verifies fix is currently holding)
                  timeframe: omit tileTimeframe; use from:now()-15m in DQL
                  note: 0 records on deploy_notebook.sh validation is EXPECTED for verification queries
                        (the incident is already resolved). The deploy script will warn but not block.
```

**Section timeframe format (--epoch true, default):**

For all non-exception DQL sections, set `state.input.timeframe` in the section JSON:
```json
"state": {
  "input": {
    "value": "<DQL with from:toTimestamp(\"WINDOW_START_ISO\"), to:toTimestamp(\"WINDOW_END_ISO\")>",
    "timeframe": { "from": "<WINDOW_START_MS>", "to": "<WINDOW_END_MS>" }
  }
}
```

For exception sections (Davis problems, related problems, verification), omit
`state.input.timeframe` entirely — the DQL carries its own relative time bounds.

**--epoch false:** omit `state.input.timeframe` from all sections; rely on `defaultTimeframe`.

**Section rules:**
- Every DQL section: `"showInput": true`, `"autoSelectVisualization": true`
  (except timeseries chart: `"autoSelectVisualization": false`, `"visualization": "areaChart"`)
- Markdown annotations MUST include:
  - What the query is showing
  - What to look for in the results
  - What "healthy" vs "unhealthy" looks like
- Skip steps 4 and 5 if no deployment events were found (mention this in the
  Step 1 markdown instead)

**4.4 — Deploy**

```bash
bash ~/.claude/skills/dt-app-notebooks/scripts/deploy_notebook.sh /tmp/<problem-id>-notebook.json
```

The script validates all DQL queries and blocks on failures.
On success it prints the notebook URL — relay that to the user.

---

## QUALITY RULES

1. **Never invent DQL field names.** If unsure, run `dtctl query` with `| limit 1`
   first to discover fields, or consult `dt-dql-essentials`.

2. **Never embed a query that hasn't been validated.** No exceptions.

3. **The ASCII diagram must reflect actual entity names and IDs from the data.**
   Do not use placeholder names.

4. **The ROOT_FAILURE_MECHANISM must be a concrete technical statement.**
   Bad: "service errors occurred". Good: "SQL Error 544 — IDENTITY_INSERT is OFF
   on table Trades; every INSERT from BrokerService is rejected by MSSQL."

5. **Markdown annotations must be written for an SRE encountering this incident
   at 3am.** Clear, actionable, no jargon beyond standard SRE vocabulary.

6. **Epoch time is the default (`--epoch true`).** Pin `defaultTimeframe` and all
   non-exception DQL sections to absolute epoch ms / ISO 8601 timestamps derived from
   the problem record. This ensures the notebook shows exactly the incident window when
   opened later — not whatever `now()-Xh` resolves to at that future time.

   **Named exceptions (always use relative time, even with `--epoch true`):**
   - `fetch dt.davis.problems` queries (Step 1.1, Step 1.4) → `from:now()-24h` — Davis
     requires relative bounds; absolute timestamps on `dt.davis.problems` are unreliable.
   - Post-fix verification query (Section 14) → `from:now()-15m` — must reflect live state
     to confirm the fix is currently holding.

   **`--epoch false`:** Switches all sections to `from:now()-Xh` relative windows.
   Use only for live incidents still in progress where the window end is unknown.

7. **The verification DQL in Remediation must be a proper `[dql]` section — never a
   markdown code fence.** An SRE must be able to click "Run" on it. Embedding DQL as
   ` ```dql ` text in a markdown section is forbidden.

8. **The verification DQL must return 0 records when the fix works.** Zero records from
   the deploy validator is expected and normal — the incident may already be resolved.
   The deploy script warns but does not block on empty-result queries.

---

## SKILLS USED BY THIS SKILL

| Skill | When used |
|---|---|
| `dt-obs-problems` | Phase 1 field names, query patterns, problem schema |
| `dt-dql-essentials` | DQL syntax, field escaping, makeTimeseries vs timeseries |
| `dt-obs-logs` | Phase 2 log queries for ERROR/AVAILABILITY categories |
| `dt-obs-tracing` | Phase 2 span queries for SLOWDOWN category |
| `dt-app-notebooks` | Phase 4 JSON structure, deploy_notebook.sh, section types |

Do NOT use `dt-rca` or `dt-rcf`.

---

## EXAMPLE OUTPUT

After successful execution the user receives:

```
✓ Problem fetched: P-260530039 — Failure rate increase
✓ Root cause: [eks-live][easytrade] TradeManagement (ERROR, 92 users)
✓ Trigger: EasyTrade 1.1.1 deployment at 18:00:24 UTC (4 min before first error)
✓ Root failure: SQL Error 544 — IDENTITY_INSERT OFF on Trades table
✓ 7/7 DQL queries validated
✓ Notebook deployed: https://demo.apps.dynatrace.com/ui/apps/dynatrace.notebooks/notebook/<id>
```
