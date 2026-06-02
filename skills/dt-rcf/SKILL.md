---
name: dt-rcf
description: >
  Root Cause Forensics (RCF) — agentic, multi-entry forensic investigation for Dynatrace.
  Unlike /dt-rca which requires a Davis Problem ID, dt-rcf accepts any forensic anchor:
  Problem ID, service name, time window, trace ID, or error pattern. Dispatches parallel
  forensic workers (dt-obs-problems, dt-obs-tracing, dt-obs-logs, dt-obs-services), builds
  a ranked hypothesis graph with confidence scores, synthesizes via Davis CoPilot, and
  generates a polished MD+PDF forensic report. Use when you need deep RCA, when Davis
  hasn't fired yet, or when the investigation spans multiple services without a clear
  Problem ID anchor.
---

# dt-rcf — Dynatrace Root Cause Forensics

Agentic forensic investigation skill. Accepts any forensic anchor, dispatches parallel
phase workers backed by maintained DQL sub-skills, builds a ranked evidence hypothesis
graph, and produces a complete forensic report (Markdown + PDF).

## Usage

```
/dt-rcf P-26012752                                    # Problem ID anchor
/dt-rcf SERVICE:"checkout-api"                        # Service name anchor
/dt-rcf FROM:"2026-05-18T14:00Z" TO:"2026-05-18T15:00Z"  # Time window anchor
/dt-rcf TRACE:"a1b2c3d4e5f6"                         # Trace ID anchor
/dt-rcf ERROR:"connection pool exhausted"             # Error pattern anchor
/dt-rcf P-26012752 -clean                             # Sanitized report
/dt-rcf SERVICE:"order-api" -clean --review           # With reviewer gate
```

## Arguments

- `$ARGUMENTS` — at least one forensic anchor (see types below)
- `-clean` — sanitize all identifying names (same rules as /dt-rca)
- `--review` — add a reviewer subagent quality gate before saving the report
- `appendix=true` — include Appendix B (Glossary) + C (Capabilities) in full; default OFF

> **Speed note:** run at **medium effort** (not high). High effort multiplies latency across all
> subagents and the report writer; the investigation quality is set by reference reads + query
> depth, not effort level.

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- dynatrace-for-ai skills installed (workers read from these as reference files):
  - `dt-obs-problems` (problem-trending, impact-analysis, problem-correlation)
  - `dt-obs-tracing` (failure-detection, sampling-extrapolation, http-spans, database-spans, rpc-spans, request-attributes, entity-lookups)
  - `dt-obs-logs`
  - `dt-obs-services` (service-metrics + runtime refs: java, nodejs, dotnet, python, php, go)
  - `dt-dql-essentials` (DQL syntax, smartscapeNodes, getNodeName signatures)
  - `dt-rca` (sanitization rules, PDF generation, Mermaid rules)
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

dt-rcf carries NO forensic DQL. Each worker reads ONE targeted reference file at start, once per subagent.

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-problems | `dt-obs-problems/references/problem-trending.md` (+ impact-analysis, problem-correlation) |
| W-tracing  | `dt-obs-tracing/references/failure-detection.md` (+ sampling-extrapolation) |
| W-logs     | `dt-obs-logs/SKILL.md` |
| W-services | `dt-obs-services/references/service-metrics.md` + runtime ref (auto-detected) |

## ENTITY MODEL (NON-NEGOTIABLE)

`classicEntitySelector()` is explicitly deprecated. Use Smartscape model throughout.

```dql
-- Filter spans to a specific service
fetch spans
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE
    | filter id == "{SERVICE_ENTITY_ID}"
    | fields id
]

-- Filter logs to a specific service
fetch logs
| filter dt.smartscape_source.id in [
    smartscapeNodes SERVICE
    | filter id == "{SERVICE_ENTITY_ID}"
    | fields id
]
```

## EXECUTION PROTOCOL

### Phase 0a: Multi-Anchor Parser

```
ANCHOR_TYPE:
  P-\d+ pattern          → PROBLEM
  SERVICE:"<name>"       → SERVICE
  FROM:"<iso>" + TO:"<iso>" → TIME_WINDOW
  TRACE:"<id>"           → TRACE
  ERROR:"<pattern>"      → ERROR_PATTERN

FLAGS:
  -clean         → CLEAN_MODE = true
  --review       → REVIEW_MODE = true
  appendix=true  → FULL_APPENDIX = true (default false)
```

### Phase 0-auth: PRE-WARM dtctl session

**FIRST action — before any data query and before any dispatch:**
```
dtctl auth refresh
```
Silent refresh of on-disk token (`DTCTL_TOKEN_STORAGE=file`). If it fails (no/expired token), ONLY THEN run once: `dtctl auth login --plain --safety-level readonly`.

**Workers never authenticate.** On `<gap: auth>`, orchestrator refreshes once and re-dispatches.

### Phase 0b: Context Bootstrap (Sequential)

**MINIMAL BOOTSTRAP — 2 queries, not 12.**

#### If ANCHOR_TYPE == PROBLEM

```dql
fetch dt.davis.problems, from:now()-24h
| filter display_id == "{PROBLEM_ID}"
| filter not(dt.davis.is_duplicate)
| fields display_id, event.name, event.category, event.status,
         event.start, event.end, root_cause_entity_id, root_cause_entity_name,
         affected_entity_ids, affected_entity_types, dt.davis.affected_users_count
```

Extract: `PROBLEM_ID`, `PROBLEM_START`, `PROBLEM_END`, `ROOT_ENTITY_ID`, `ROOT_ENTITY_NAME`, `AFFECTED_ENTITY_IDS[]`, `ROOT_SERVICE_ID`, `ENTITY_TYPE`, `INVESTIGATION_STRATEGY`, `IS_K8S_WORKLOAD`, `K8S_NAMESPACE`, `K8S_WORKLOAD`, `RUNTIME`

**Q2 — combined ROOT_PGI + k8s + runtime (ONE query — REQUIRED):**
```dql
fetch spans, from:now()-{SCAN_DAYS}d
| filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
| summarize spancount = count(),
            by: { dt.entity.process_group_instance, k8s.namespace.name, k8s.workload.name }
| sort spancount desc
| limit 1
```

Set `ROOT_PGI` = `dt.entity.process_group_instance` (NEVER a `PROCESS_GROUP-…` id).

DERIVED_TIME_WINDOW:
```
from: PROBLEM_START minus 2h
to:   PROBLEM_END plus 1h (or now() if still ACTIVE)
```

#### If ANCHOR_TYPE == SERVICE

```
Tool: mcp__dynatrace__find_entity_by_name
Input: SERVICE_NAME from ANCHOR_VALUE
Extract: ROOT_SERVICE_ID (SERVICE-XXXX format), ROOT_ENTITY_NAME
DERIVED_TIME_WINDOW: FROM/TO from arguments, or now()-2h to now()
INVESTIGATION_STRATEGY = "open"
```

#### If ANCHOR_TYPE == TRACE

```dql
fetch spans, from:now()-24h
| filter trace.id == toUid("{ANCHOR_VALUE}")
| fieldsAdd service_name = getNodeName(dt.smartscape.service)
| limit 1
| fields trace.id, dt.smartscape.service, service_name, start_time
```

#### If ANCHOR_TYPE == TIME_WINDOW
```
DERIVED_TIME_WINDOW from FROM/TO arguments directly
ROOT_SERVICE_ID = null (W-tracing will surface top error services in window)
INVESTIGATION_STRATEGY = "open"
```

#### If ANCHOR_TYPE == ERROR_PATTERN
```dql
fetch logs, from:now()-4h
| filter matchesPhrase(content, "{ANCHOR_VALUE}")
| filter loglevel == "ERROR"
| summarize count = count(), by: {dt.smartscape_source.id}
| sort count desc | limit 1
```

### Phase 0c: Reference Strategy

dt-rcf is **reference-driven** — workers carry no hardcoded forensic DQL. Each worker reads ONE targeted reference file at start.

### Phase 1: Parallel Worker Dispatch

**Dispatch ALL workers in ONE message with multiple Agent calls — true concurrency.**

Workers P, T, L, S run concurrently as subagents (`subagent_type: Explore`).
Skip W-problems if ANCHOR_TYPE != PROBLEM.

#### Worker Rules (ALL workers)

```
STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed. Read ONCE. Take query patterns from it;
apply DISPATCH CONTEXT scoping. This is the ONLY source of DQL truth.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple dtctl query tool calls).

ENTITY MODEL — CRITICAL:
- NEVER use dt.entity.service/host == "..." to filter spans or logs.
  Use: filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "ID" | fields id]
- NEVER use classicEntitySelector().
- Process/runtime metrics: scope by dt.entity.process_group_instance == "{ROOT_PGI}"
  NEVER filter on dt.process_group.id (frequently EMPTY — yields false 0-row results).
- For entity names: getNodeName(dt.smartscape.service)

RULES:
- PREFLIGHT: run DTCTL_TOKEN_STORAGE=file dtctl query 'fetch spans | limit 1'
  If auth/connection error, return <gap: dtctl-unavailable: {message}> and STOP.
- Do NOT return raw rows. Internalize; summarize aggressively.
- Cap PhaseResult at ~4KB. Top-5 of anything, not top-50.
- AUTH IS THE ORCHESTRATOR'S JOB — workers NEVER run dtctl auth login.
  On auth error, return <gap: auth>.
- ERROR ≠ ABSENCE: a query that ERRORED tells you nothing about whether data exists.
- PROOF OF ABSENCE (required): if you conclude "not present", include the exact
  UNFILTERED confirmation query and its zero-row result.
- Execute DQL via DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'
  SINGLE QUOTES around the DQL. NO BACKTICKS. Alias every bin.
```

#### W-problems PhaseResult shape
```json
{
  "problem_summary": { "category": "...", "status": "...", "duration_min": 0 },
  "recurrence_count_30d": 0,
  "typical_duration_min": 0,
  "blast_radius": { "entity_count": 0, "user_count": 0 },
  "affected_entity_names": [{ "id": "...", "name": "..." }],
  "deployment_in_window": { "found": false, "minutes_before_problem": null, "description": null },
  "gaps": []
}
```

#### W-tracing PhaseResult shape
```json
{
  "failure_reasons": { "http_code": 0, "exception": 0, "grpc_code": 0, "span_status": 0, "custom_rule": 0 },
  "top_exceptions": [{ "type": "...", "count": 0, "sample_trace_id": "..." }],
  "failing_paths": [{ "endpoint": "...", "status_code": 0, "count": 0 }],
  "request_count_raw": 0,
  "request_count_extrapolated": 0,
  "sampling_note": "1 span = ~N requests",
  "outbound_degradation": [{ "span_name": "...", "avg_ms_baseline": 0, "avg_ms_peak": 0 }],
  "gaps": []
}
```

#### W-logs PhaseResult shape
```json
{
  "top_exceptions": [{ "message": "...", "count": 0 }],
  "timeout_count": 0,
  "oom_count": 0,
  "log_error_rate_trend": [{ "time_bucket": "...", "error_rate_pct": 0 }],
  "log_inflection_time": "ISO or null",
  "gaps": []
}
```

#### W-services PhaseResult shape
```json
{
  "error_rate_pct_peak": 0,
  "p95_ms_peak": 0,
  "p99_ms_peak": 0,
  "p95_ms_baseline": 0,
  "runtime": "java|nodejs|...",
  "runtime_signals": [{ "metric": "...", "finding": "...", "severity": "high|medium|low" }],
  "gaps": []
}
```

**CRITICAL for W-logs scoping (k8s fallback required):**
```dql
filter dt.smartscape_source.id in [smartscapeNodes SERVICE | filter id=="{ROOT_SERVICE_ID}" | fields id]
   or dt.source_entity in [ …same… ]
   or (k8s.namespace.name=="{K8S_NAMESPACE}" and k8s.container.name=="{K8S_WORKLOAD}")
```
The k8s clause is REQUIRED — the Smartscape source filter returns 0 for OTel services.

**CRITICAL for W-services (span-derived RED, NOT dt.service.request.*):**
```dql
fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
| filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
| filter request.is_root_span == true
| summarize reqs=count(), fails=countIf(request.is_failed==true),
            p95_ms=percentile(duration,95)/1e6, p99_ms=percentile(duration,99)/1e6
| fieldsAdd err_pct=(fails*100.0)/reqs
```

### Phase 1.5: Absence Gate

Before using any worker result, scan every PhaseResult for absence claims. For EACH:
1. Require worker's proof (unfiltered confirmation query + zero-row result)
2. Run ONE cheap unfiltered confirmation query yourself
3. Only after confirmation may "absent" enter the report

### Phase 2: Hypothesis Graph Construction

Build ranked hypotheses:
```json
{
  "hypothesis": "<concrete mechanism>",
  "evidence_refs": [
    { "phase": "T", "sub_skill": "dt-obs-tracing", "finding": "..." }
  ],
  "confidence_factors": {
    "span_evidence":     0.0-1.0,   // weight: 0.35
    "log_corroboration": 0.0-1.0,  // weight: 0.25
    "runtime_signal":   0.0-1.0,   // weight: 0.25
    "prior_recurrence": 0.0-1.0    // weight: 0.15
  },
  "confidence": 0.0-1.0
}
```

Sort by `confidence` descending. H1 is primary hypothesis.
- ≥ 0.80 → High confidence
- 0.50-0.79 → Medium
- < 0.50 → Low / contributing factor

### Phase 3: Davis CoPilot Synthesis

Pass ranked hypothesis graph (NOT raw findings) to Davis CoPilot. Ask for validation, remediation, and what additional data would increase confidence.

### Phase 4: Report Generation

Report sections (emit in order):
1. H1 title + H2 subtitle (Investigation {ANCHOR}: {ROOT_ENTITY_NAME} {CATEGORY})
2. Header: Generated / Analyst / Environment / Anchor / Window / Phases
3. Executive Summary: Critical Findings, Impact Summary, Business Impact, Immediate Action, Event Timeline (gantt)
4. Key Talking Points
5. Forensic Hypothesis Rankings table
6. Topology & Architecture (service dependency map from Smartscape)
7. Sequence of Events
8. Root Cause Analysis: Summary, Decisive Evidence (3-5 row extract), Failure Cascade, Contributing Factors, Recurrence
9. Forensic Analysis: Failure Taxonomy, Evidence Chain, Runtime Findings
10. Recommended Resolution
11. Appendix A: Investigation Details
12. Appendix B: Glossary (only if FULL_APPENDIX)
13. Appendix C: AI Capabilities (only if FULL_APPENDIX)
14. Appendix D: Query Exchange Log (full if FULL_APPENDIX, else log pointer only)
15. Links + *End of Report*

**Filenames:**
```
Normal: RCF_{ANCHOR_SLUG}_{DATE}.md / .pdf
Clean:  RCF_{ANCHOR_SLUG}_{DATE}_SANITIZED.md / .pdf
```

## RULES (NON-NEGOTIABLE)

1. **Smartscape-native entity model** — NEVER `fetch dt.entity.*` for SERVICE, HOST, CLOUD_APPLICATION lookups. NEVER `filter dt.entity.service/host ==` on spans/logs/events.
2. **Sub-skills are DQL authority** — never hardcode a DQL pattern that exists in a sub-skill reference.
3. **`not(dt.davis.is_duplicate)` required** — on ALL `fetch dt.davis.problems` queries.
4. **Failure taxonomy required** — W-tracing must attempt `dt.failure_detection.results[][reason]` breakdown.
5. **Sampling extrapolation required** — report both raw span count and extrapolated request count.
6. **Hypothesis graph required** — Phase 2 must produce ranked hypotheses before Davis CoPilot.
7. **Evidence chain required** — cite phase + sub-skill source for every finding in RCA.
8. **Parallel worker dispatch in single message** — ALL workers dispatched in ONE message.
9. **Error/empty ≠ absence** — verify before concluding.
10. **dtctl auth: on-disk tokens, lazy login, orchestrator-only** — Workers NEVER log in.
11. **Mermaid in MD, ASCII in PDF** — use two-file strategy (`_pdf.md` intermediate).
12. **No DQL in body** — only in Appendix D.

## MERMAID SYNTAX RULES

- Gantt task names must contain NO colon (`:` is the delimiter)
- Sequence message labels: no literal double-quotes and no second colon
- Graph node labels: use `<br/>` not `\n`
- No raw `file:line` in labels — use `file Lnn`

## PDF DIAGRAM RULES

No Mermaid in PDFs — ASCII art only, inside plain code blocks, 80-char max width.
Create `_pdf.md` intermediate, convert ALL Mermaid to ASCII, run `md-to-pdf`, delete intermediate.
