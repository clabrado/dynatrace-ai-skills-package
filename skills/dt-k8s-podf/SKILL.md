---
name: dt-k8s-podf
description: >-
  Pod Debug Forensics — "kubectl describe on steroids" for Dynatrace. Given a
  Kubernetes namespace + workload (and optionally a specific pod), runs a parallel
  forensic investigation across K8s events, container logs, distributed traces, and
  applied network policies, classifies the root cause (OOMKilled, CrashLoopBackOff,
  ImagePullBackOff, probe failure, denied egress, DNS resolution failure, app
  exception, scheduling/resource-pressure, config drift), and produces either a
  polished Markdown forensic report (default; PDF optional via `--pdf`) or a
  deployable Dynatrace Notebook
  (`--notebook`). Composes dt-obs-kubernetes, dt-obs-logs, dt-obs-tracing, and —
  in notebook mode — dt-pr-notebooks. Use when a pod is crashing, restarting,
  failing readiness/liveness probes, throwing app errors that don't match logs,
  experiencing OOM kills, stuck in ImagePullBackOff/CrashLoopBackOff, hitting a
  denied NetworkPolicy egress, or behaving like "phantom errors" with no obvious
  cause in the workload itself. NEVER uses dt-rca or dt-rcf.
---

# dt-k8s-podf — Kubernetes Pod Debug Forensics

Agentic, parallel forensic skill for a single Kubernetes workload. Anchors on
`NS:"<namespace>" WORKLOAD:"<workload>" [POD:"<pod-name>"]`, fans out four
parallel workers (events, logs, traces, network), synthesizes a typed root cause
classification, asks Davis CoPilot for a cause-class-specific remediation
runbook, then ships either a Markdown forensic report (default; PDF optional via
`--pdf`) or a deployed Dynatrace Notebook (`--notebook`).

This is the substrate `/dt-rcf` pattern adapted to the Kubernetes pod-debug
domain. The four sub-domains it covers — K8s events, container logs, span
errors, and applied NetworkPolicies — together replace what an SRE would
otherwise piece together from `kubectl describe pod`, `kubectl logs --previous`,
`kubectl get events`, and three browser tabs of trace/network views.

---

## Usage

```
/dt-k8s-podf NS:"checkout" WORKLOAD:"checkout-api"
/dt-k8s-podf NS:"payments" WORKLOAD:"payment-worker" POD:"payment-worker-7d4f9-xkj2p"
/dt-k8s-podf NS:"frontend" WORKLOAD:"web-ui" --notebook
/dt-k8s-podf NS:"orders" WORKLOAD:"order-svc" -clean
/dt-k8s-podf NS:"trades" WORKLOAD:"broker-svc" --notebook -clean
/dt-k8s-podf NS:"checkout" WORKLOAD:"checkout-api" --pdf   # also render a PDF alongside the .md
```

> **Output:** the canonical deliverable is **Markdown only**. PDF is opt-in via
> `--pdf` and is non-fatal — if the PDF engine is unavailable the run logs one
> line and still produces the `.md`. `--notebook` is a separate mode that
> deploys a Dynatrace Notebook instead of a report (PDF flag does not apply).

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `NS:"<namespace>"` | yes | Kubernetes namespace (`k8s.namespace.name`) |
| `WORKLOAD:"<workload>"` | yes | Kubernetes workload name (`k8s.workload.name`) — Deployment / StatefulSet / DaemonSet / Job |
| `POD:"<pod-name>"` | optional | Narrow scope to a single `k8s.pod.name`; omit to investigate the workload as a whole |
| `--notebook` | optional | Deploy a Dynatrace Notebook (via dt-pr-notebooks substrate) instead of a Markdown report |
| `--pdf` | optional | Also render a PDF alongside the Markdown report. **Default OFF** (MD-only). Legacy `pdf=true` is accepted as an alias; `pdf=false` is a no-op. Non-fatal: a missing PDF engine logs one line and the run still completes with the `.md`. Ignored in `--notebook` mode. |
| `-clean` | optional | Sanitize all identifying names (same rules as `/dt-rca` Phase 1.14) |
| `appendix=true` | optional | Include Appendix B (Glossary) + C (Capabilities) in full; default OFF |

> **Speed note:** Run at **medium** effort. High effort multiplies latency across
> all four workers without improving findings — investigation quality is set by
> reference reads + query coverage, not effort level.

---

## Sub-Skills Loaded Per Phase

dt-k8s-podf carries NO domain DQL of its own. Each worker reads ONE targeted
reference file (its authority) at start, derives queries from it, then applies
the dt-k8s-podf scoping below.

| Worker | Reference file(s) (read ONCE at worker start) |
|---|---|
| W-events | `~/.agents/skills/dt-obs-kubernetes/references/workload-health.md` (+ `pod-debugging.md` for OOM/probe/eviction patterns) |
| W-logs   | `~/.agents/skills/dt-obs-logs/SKILL.md` |
| W-traces | `~/.agents/skills/dt-obs-tracing/references/failure-detection.md` (+ `http-spans.md` / `rpc-spans.md` / `database-spans.md` based on detected protocol) |
| W-network | `~/.agents/skills/dt-obs-kubernetes/references/network-policies.md` (+ `labels-annotations.md` for owner/team resolution) |

**Inheritance:** Phase 1.14 sanitization, all PDF generation, Mermaid syntax,
diagram sizing, and dtctl auth hygiene are **identical to `/dt-rca` and
`/dt-rcf`** — read `~/.claude/skills/dt-rca/SKILL.md` sections "Phase 1.14",
"PDF DIAGRAM RULES", "MERMAID SYNTAX RULES", and "DIAGRAM SIZING RULES" if you
need the full text. For `--notebook` mode, also inherit the notebook JSON
authoring + deploy pattern from
`~/.claude/skills/dt-pr-notebooks/SKILL.md` Phase 4.

---

## Skill Registry

Maintenance-time reference map — the authoritative source for where each inline
query came from. Used when updating / re-validating the skill. NOT read at run
time.

### DQL Authority

| File | Purpose | When to read |
|------|---------|--------------|
| `~/.agents/skills/dtctl/references/DQL-reference.md` | Core DQL syntax, filter patterns | Always (Phase 0c) |
| `~/.agents/skills/dt-dql-essentials/references/dql/dql-functions-smartscape.md` | smartscapeNodes / getNodeName / getNodeField | Always (Phase 0c) |

### W-events — dt-obs-kubernetes

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-kubernetes/SKILL.md` | Always |
| `~/.agents/skills/dt-obs-kubernetes/references/workload-health.md` | Always — Deployment/StatefulSet/DaemonSet health, restart trends |
| `~/.agents/skills/dt-obs-kubernetes/references/pod-debugging.md` | Always — OOMKilled, CrashLoopBackOff, probe failures, eviction reasons |
| `~/.agents/skills/dt-obs-kubernetes/references/labels-annotations.md` | Always — owner/team resolution via labels |
| `~/.agents/skills/dt-obs-kubernetes/references/pod-node-placement.md` | If FailedScheduling events appear |

### W-logs — dt-obs-logs

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-logs/SKILL.md` | Always |

### W-traces — dt-obs-tracing

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-tracing/SKILL.md` | Always |
| `~/.agents/skills/dt-obs-tracing/references/failure-detection.md` | Always — failure reason taxonomy, exception sources |
| `~/.agents/skills/dt-obs-tracing/references/http-spans.md` | If sample span shows `http.request.method` |
| `~/.agents/skills/dt-obs-tracing/references/rpc-spans.md` | If sample span shows `rpc.system` |
| `~/.agents/skills/dt-obs-tracing/references/database-spans.md` | If sample span shows `db.system` (outbound dep slowness) |

### W-network — dt-obs-kubernetes

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-kubernetes/references/network-policies.md` | Always — applied policies, deny events |
| `~/.agents/skills/dt-obs-kubernetes/references/ingress.md` | If workload is Ingress-fronted (detected via labels) |

---

## ENTITY MODEL — SMARTSCAPE-NATIVE (NON-NEGOTIABLE)

Same model as `/dt-rcf`. K8s adds one important pattern: **scope by
`k8s.namespace.name` + `k8s.workload.name`** on every events/logs/spans query.
This is the only k8s domain where the namespace/workload pair is the canonical
forensic scope — the Smartscape SERVICE filter alone misses sidecars, init
containers, and pods that haven't yet attached to a service entity.

### K8s Anchor Filter (every events/logs/spans query)

```dql
| filter k8s.namespace.name == "{NS}"
| filter k8s.workload.name  == "{WORKLOAD}"
[ | filter k8s.pod.name      == "{POD}" ]   -- only if POD: provided
```

### Span Filtering (workload-scoped)

```dql
-- All spans originating in the workload
fetch spans
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| fieldsAdd service_name   = getNodeName(dt.smartscape.service),
            container_name = k8s.container.name
```

### Cloud Application + Workload Metadata Resolution

```dql
-- Cloud application entity for the workload (drives owner labels, cluster ID)
fetch dt.entity.cloud_application
| filter entity.name == "{WORKLOAD}"
| filter contains(toString(tags), "{NS}")
| fieldsAdd entity.name, tags, belongs_to[dt.entity.kubernetes_cluster],
            metadata.labels
| limit 1
```

`labels-annotations.md` is the authority for parsing `metadata.labels` into
owner / team / commit-sha / app-version.

### Container Discovery (drives W-logs / W-traces)

```dql
-- Containers running in the workload (for log filter array + sidecar detection)
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize span_count = count(), by: { k8s.container.name }
| sort span_count desc
```

If the workload has no spans (jobs, sidecars without instrumentation), fall
back to logs:

```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize log_count = count(), by: { k8s.container.name }
| sort log_count desc
```

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking — set up at the very start)

Identical pattern to dt-rcf. Set `LOG="PODF_{NS}_{WORKLOAD}_{DATE}.log"` and at
each phase boundary append ONE line backgrounded:
```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line>" >> "$LOG"; } &
```
Log: START, Phase 0b.0 Substrate Probe summary (event count at default window +
auto-widen decision + resolved event-type set), Phase 0b complete (DispatchContext
summary), Phase 1 dispatch, Phase 1 complete (per-worker one-liners), Absence-Gate
result, Phase 2 cause classification, report/notebook written, RUN COMPLETE.
Phase 7 prints the log path.

---

### Phase 0a: Argument Parser

```
NS         = value after  NS:"<...>"        (REQUIRED)
WORKLOAD   = value after  WORKLOAD:"<...>"  (REQUIRED)
POD        = value after  POD:"<...>"       (OPTIONAL)
NOTEBOOK   = true if --notebook present
PDF_MODE   = true if --pdf present (alias: pdf=true). pdf=false is a no-op.
             DEFAULT FALSE — Markdown is the canonical deliverable.
             Ignored when NOTEBOOK == true (notebook mode emits no PDF).
CLEAN_MODE = true if -clean present
FULL_APPENDIX = true if appendix=true

VALIDATION:
  - NS and WORKLOAD must both be present and non-empty
  - WORKLOAD must match Kubernetes naming rules (lowercase RFC 1123)
  - On invalid input → print usage and exit
```

---

### Phase 0-auth: PRE-WARM the dtctl session

**FIRST action of the whole run — before any data query and before any dispatch:**
```bash
dtctl auth refresh
```
This is a REFRESH, not a login. Workers inherit the on-disk token; they never
authenticate themselves. If `dtctl auth refresh` errors with no/expired refresh
token (first-ever use, or weeks idle), ONLY THEN run once:
```bash
dtctl auth login --plain --safety-level readonly
```

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

This is identical to dt-rcf Phase 0-auth — read that section if you need the
full rationale.

---

### Phase 0b: Workload Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** for the four phase workers.

**MINIMAL BOOTSTRAP — Phase 0b.0 Substrate Probe + 3 bootstrap queries.**
Bootstrap is the orchestrator's only serial stretch — keep it tiny.

#### Phase 0b.0 — Substrate Probe (run FIRST, before Q1–Q3)

The orchestrator must NOT assume which K8s event substrate exists or that the
default 6h window has any events. Run ONE cheap probe to discover the real shape
at runtime, then commit to a window and an event-type candidate set. This is the
self-healing step — it directly drives the PF-K1 auto-widen and the PF-K2
broadened cause-class triggers below.

> **Why probe:** Tenant validation 2026-06-03 against
> `NS:"unguard" WORKLOAD:"unguard-user-simulator"` proved the default 6h window
> returns **0 events** while 72h returns **7,920** — all surfaced as
> `event.kind == "DAVIS_EVENT"` with `event.type ∈ {CUSTOM_INFO, ERROR_EVENT,
> WARNING}`. On the bulk of these events `reason` / `event.description` are
> **NULL** and the diagnostic phrase lives in `event.name` (e.g. `"Back-off
> restarting failed container …"`). Measured delta at 72h: the legacy
> `reason`/`event.description`-only counters read `crash_loop=1146`,
> `restarts=9`; the broadened `event.name`-aware counters read
> `crash_loop=4875`, `restarts=3737` — a ~400× undercount on restarts. At the
> default 6h window the legacy classifier reads **0** and falls through to
> `inconclusive` even though the workload is crash-looping.

**Probe P0 — event presence at the default window, by type:**

<!-- VALIDATED 2026-06-03 against live tenant (unguard/unguard-user-simulator):
     6h → 0 rows; 72h → CUSTOM_INFO 5243 / ERROR_EVENT 1549 / WARNING 1117 -->
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize c = count(), by: { event.kind, event.type }
| sort c desc
| limit 20
```

**Probe decision logic (binds variables the later phases interpolate):**

```
DEFAULT_SCAN_HOURS  = SCAN_HOURS         # capture the default (6) BEFORE any widen
EVENT_COUNT_DEFAULT = sum of c across all rows from Probe P0

# --- PF-K1 AUTO-WIDEN ---
IF EVENT_COUNT_DEFAULT == 0:
    Re-run Probe P0 ONCE with from:now()-72h.
    EVENT_COUNT_WIDE = sum of c across rows at 72h.
    IF EVENT_COUNT_WIDE > 0:
        SCAN_HOURS    = 72            # widen ALL downstream phase queries
        WINDOW_WIDENED = true         # raise the auto-widen banner (Phase 2 / report)
    ELSE:
        WINDOW_WIDENED = false        # genuinely empty even at 72h → real-absence path
                                      # (still gated by Phase 1.5 Absence Gate)
ELSE:
    SCAN_HOURS     = {default}        # 6h had events — keep it
    WINDOW_WIDENED = false

# --- PF-K2 EVENT-TYPE CANDIDATE SET (from the resolved window) ---
PRESENT_EVENT_TYPES = distinct event.type values returned by the winning Probe P0
                      run (the one with rows).
EVENT_KIND          = distinct event.kind value(s) — typically "DAVIS_EVENT" on
                      this tenant family (NO dt.system.bucket filter — see Rule 2).

# CANDIDATE LIST, ordered [corrected-2026-06-03, legacy]:
#   Davis event types (corrected, present today): CUSTOM_INFO, ERROR_EVENT, WARNING
#   kubelet reasons (legacy, may be empty here):  OOMKilled, BackOff, ImagePullBackOff,
#                                                 FailedScheduling, Evicted
# Bind DAVIS_TYPES_PRESENT = PRESENT_EVENT_TYPES ∩ {CUSTOM_INFO, ERROR_EVENT, WARNING}
# so W-events' typed counters classify whichever substrate actually exists.
```

Emit a one-line probe summary to the run log, e.g.:
`Probe 0b.0: 6h=0 events → AUTO-WIDEN to 72h (7920 events); kind=DAVIS_EVENT; types=CUSTOM_INFO,ERROR_EVENT,WARNING; reason field NULL → classify on event.type + event.name`

If BOTH the 6h and 72h probe runs return zero rows, do NOT widen blindly further
— set `WINDOW_WIDENED=false`, leave `SCAN_HOURS` at the (widened-attempt) value,
and let the existing Phase 1.5 Absence Gate confirm genuine absence before any
"no events" claim enters the report. Absence is gated, never assumed.

Store `SCAN_HOURS`, `WINDOW_WIDENED`, `PRESENT_EVENT_TYPES`, `EVENT_KIND`, and
`DAVIS_TYPES_PRESENT` into DispatchContext — Q2 and every phase worker use the
resolved `SCAN_HOURS`, and W-events uses the event-type set.

#### Q1 — Cloud application + owner labels

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch dt.entity.cloud_application, from:now()-7d
| filter entity.name == "{WORKLOAD}"
| filter contains(toString(tags), "{NS}")
| fieldsAdd entity.name, tags, belongs_to[dt.entity.kubernetes_cluster],
            metadata.labels, metadata.annotations,
            kubernetesAnnotations, kubernetesLabels
| limit 1
```

Extract:
```
CLOUD_APP_ID       id of the cloud_application entity
CLUSTER_ID         belongs_to[dt.entity.kubernetes_cluster] (first element)
OWNER_TEAM         metadata.labels["owner"] or metadata.labels["team"]
                   or metadata.annotations["dynatrace.com/team"]
DEPLOYMENT_KIND    inferred from labels (e.g. "app.kubernetes.io/component")
APP_VERSION        metadata.labels["app.kubernetes.io/version"]
```

If Q1 returns 0 rows, set `CLOUD_APP_ID = null` and continue — many workloads
(Jobs, CronJobs, sidecars) don't have a registered cloud_application. The
workers can still investigate by namespace+workload scope.

#### Q2 — Container discovery + workload health snapshot

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
<!-- Uses the SCAN_HOURS resolved by Phase 0b.0 (post-auto-widen), not a literal 6h -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize span_count   = count(),
            error_count  = countIf(request.is_failed == true),
            first_seen   = min(start_time),
            last_seen    = max(start_time),
            by: { k8s.container.name }
| sort span_count desc
| limit 10
```

Extract:
```
CONTAINERS[]       array of k8s.container.name values from rows
PRIMARY_CONTAINER  row with highest span_count (skips istio-proxy if app sibling exists)
SPAN_PRESENCE      true if any rows returned
```

If `SPAN_PRESENCE == false`, run the logs fallback:

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
<!-- Uses the SCAN_HOURS resolved by Phase 0b.0 (post-auto-widen), not a literal 6h -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize log_count = count(), by: { k8s.container.name }
| sort log_count desc
| limit 10
```

If both return zero rows: this is a real-absence finding — the workload either
doesn't exist in this tenant, the namespace is misspelled, or the pod has been
gone long enough to drop out of retention. Document and continue with empty
containers (W-events can still report on K8s events).

#### Q3 — Cluster name resolution

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
smartscapeNodes KUBERNETES_CLUSTER
| filter id == "{CLUSTER_ID}"
| fields id, name
```

Set `CLUSTER_NAME` for report headers. Skip if `CLUSTER_ID == null`.

#### DERIVED_TIME_WINDOW

```
SCAN_HOURS = 6   (DEFAULT — most pod debug investigations are recent)
                 If POD is provided and the pod no longer exists, widen to 24.

OVERRIDE (PF-K1 auto-widen): Phase 0b.0's Substrate Probe may have already
reset SCAN_HOURS to 72 because the 6h window held 0 events. ALWAYS use the
SCAN_HOURS value that Phase 0b.0 resolved — do not reset it back to 6 here.
If WINDOW_WIDENED == true, every downstream phase query and the report header
must use 72h, and the report/run-log must carry the auto-widen banner.

from: now() - SCAN_HOURS hours
to:   now()
```

Store as **DispatchContext JSON**:

```json
{
  "NS": "...",
  "WORKLOAD": "...",
  "POD": "... or null",
  "CLOUD_APP_ID": "CLOUD_APPLICATION-XXXX or null",
  "CLUSTER_ID": "KUBERNETES_CLUSTER-XXXX or null",
  "CLUSTER_NAME": "... or null",
  "OWNER_TEAM": "... or null",
  "DEPLOYMENT_KIND": "Deployment | StatefulSet | DaemonSet | Job | CronJob | unknown",
  "APP_VERSION": "... or null",
  "CONTAINERS": ["app", "istio-proxy", ...],
  "PRIMARY_CONTAINER": "...",
  "SPAN_PRESENCE": true,
  "DERIVED_TIME_WINDOW": { "from": "now()-{SCAN_HOURS}h", "to": "now()" },
  "SCAN_HOURS": 6,
  "WINDOW_WIDENED": false,
  "EVENT_KIND": "DAVIS_EVENT",
  "PRESENT_EVENT_TYPES": ["CUSTOM_INFO", "ERROR_EVENT", "WARNING"],
  "DAVIS_TYPES_PRESENT": ["CUSTOM_INFO", "ERROR_EVENT", "WARNING"],
  "NOTEBOOK_MODE": false,
  "PDF_MODE": false,
  "CLEAN_MODE": false
}
```

> `SCAN_HOURS` here reflects the value AFTER Phase 0b.0 resolution (6 by default,
> 72 if auto-widened). `WINDOW_WIDENED`, `EVENT_KIND`, `PRESENT_EVENT_TYPES`, and
> `DAVIS_TYPES_PRESENT` are produced by the Substrate Probe and consumed by
> W-events (typed counters) and Phase 2 (cause classification + banner).

---

### Phase 0c: Reference Strategy

dt-k8s-podf is **reference-driven** — workers carry no hardcoded DQL. Each
worker reads ONE (or two, for protocol-specific tracing) targeted reference
file at start, derives the query patterns from it, and applies the
dt-k8s-podf scoping (NS / WORKLOAD / POD filter; logs on `timestamp`; spans
on `start_time`; aliased `bin()`).

**Orchestrator exceptions (inline only):** Phase 0b bootstrap (Q1–Q3) and
Phase 1.5 Absence-Gate confirmation queries remain inline.

Proceed to Phase 1.

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

**Dispatch ALL workers in ONE message with multiple Agent calls.** Workers W-events,
W-logs, W-traces, W-network run concurrently as subagents (`subagent_type: Explore`).

Skip W-traces if `SPAN_PRESENCE == false` AND no instrumentation is expected
(documented in the report as "workload not instrumented — Job/CronJob or no
OneAgent"). Always run W-events, W-logs, and W-network regardless.

**Speed model:** each worker reads ONE targeted reference file once, fires its
queries in parallel. Expected total: bootstrap + worker fan-out (parallel) +
synthesis ≈ 6–8 min at medium effort.

**Worker-failure handling — NO silent serial fallback.** Re-dispatch the failed
worker ONCE as a fresh Agent call; if it still fails, record a `gap` for that
phase in the report. The only queries the orchestrator runs itself are
Phase 0b bootstrap and Phase 1.5 Absence-Gate confirmations.

#### Worker Prompt Template (shared header — all 4 workers)

```
You are a Kubernetes pod-debug forensic worker for dt-k8s-podf.

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed in REFERENCE FILES. Read each file ONCE. Do
not read any other files. Take the query patterns from the reference; apply
the DispatchContext scoping. This is the ONLY source of DQL truth — do NOT
derive queries from training knowledge or guess field names.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple `dtctl query` tool
calls, in parallel).

K8S SCOPING — REQUIRED on every events/logs/spans query:
  | filter k8s.namespace.name == "{NS}"
  | filter k8s.workload.name  == "{WORKLOAD}"
  [ | filter k8s.pod.name      == "{POD}" ]   -- only if POD provided

ENTITY MODEL:
- Logs filter on `timestamp` (NOT start_time)
- Spans filter on `start_time`
- Events filter on `timestamp`. **Do NOT add `dt.system.bucket` filter** —
  tenant validation 2026-05-28 confirmed K8s events surface as
  `event.kind == "DAVIS_EVENT"` without a bucket constraint on this tenant
  family; adding it returns zero rows.
- NEVER use `filter dt.entity.service ==` to scope spans/logs/events — use the
  k8s.namespace.name + k8s.workload.name pair, or dt.smartscape.* fields
- For Smartscape names: getNodeName(dt.smartscape.service)
- ALIAS every bin (`ts = bin(timestamp, 5m) | sort ts asc`); NEVER backticks
- Single-quote the dtctl arg; prefix EVERY call with DTCTL_TOKEN_STORAGE=file

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

REFERENCE FILES:
{worker-specific list — see below}

QUERIES TO RUN:
{worker-specific instructions — see below}

RULES:
- EXPLORATION CAP: run ONLY the queries defined below. No "let me also check…".
  If a finding requires one additional query, derive it from your already-loaded
  reference — never a free guess.
- Cap PhaseResult at ~4KB. Top-5 of anything, not top-50.
- Auth errors → workers NEVER run `dtctl auth login` OR `dtctl auth refresh`;
  return `<gap: auth>` and STOP — never attempt any auth recovery. Orchestrator
  re-auths and re-dispatches. Auth errors are NEVER evidence of absence.
- Error/empty ≠ absence — confirm any "no events / no logs / no policies"
  finding with a broader unfiltered query (drop the WORKLOAD filter; keep NS)
  before claiming absence. Attach the confirmation query + zero-row result
  as PROOF in your PhaseResult.

RETURN only the PhaseResult JSON schema (no prose). Nothing else.
```

---

#### W-events — Kubernetes Events Worker

**REFERENCES:** `~/.agents/skills/dt-obs-kubernetes/references/workload-health.md`
and `pod-debugging.md`. Also read `labels-annotations.md` for owner enrichment
if `OWNER_TEAM == null` in DispatchContext.

**Queries to run (ALL in one parallel batch):**

> **Tenant validation 2026-05-28:** Do NOT add `dt.system.bucket` filter — K8s
> events are surfaced as `event.kind == "DAVIS_EVENT"` without a bucket
> constraint on this tenant family. The previous `dt.system.bucket ==
> "default_events"` filter returned zero rows. Workload-scoping via
> `k8s.namespace.name` + `k8s.workload.name` is sufficient.
>
> **Tenant validation 2026-06-03 (PF-K2 — broadened classification):** On this
> tenant family the kubelet `reason` field is **NULL** and `event.description`
> is empty; the diagnostic phrase lives in **`event.name`** (e.g. `"Back-off
> restarting failed container unguard-user-simulator in pod …"`), and events
> carry Davis `event.type` ∈ `{CUSTOM_INFO, ERROR_EVENT, WARNING}`. A counter
> keyed only on `reason ==`/`event.description` reads **0 even when thousands of
> events exist**. The typed-counter query below therefore matches phrases across
> `event.name` **AND** `event.description` **AND** `reason`, and ALSO tallies the
> Davis `event.type` set so the classifier never reads 0 when events are present.
> Use the `SCAN_HOURS` and `DAVIS_TYPES_PRESENT` resolved by Phase 0b.0.

**1) All workload events (last `SCAN_HOURS`):**

<!-- VALIDATED 2026-05-28 against live tenant — no dt.system.bucket filter -->
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| fields ts = timestamp, event.type, event.kind, event.name,
         event.description, k8s.pod.name, k8s.container.name, reason
| sort ts desc
| limit 200
```

> **Read `event.name` first.** On this tenant family `reason` and
> `event.description` are typically NULL — the human-readable diagnostic (e.g.
> `"Back-off restarting failed container …"`) is in `event.name`. Build the
> Event Timeline from `event.name` when `reason`/`event.description` are empty.

**2) Restart + OOM + probe-failure breakdown (typed counters — PF-K2 broadened):**

> **PF-K2:** each phrase counter matches across `event.name` AND
> `event.description` AND `reason` — because on this tenant family `reason` and
> `event.description` are NULL and the diagnostic phrase lives in `event.name`
> (e.g. `"Back-off restarting failed container …"`). The trailing
> `davis_error/warning/custom_info` counters tally the Davis `event.type` set so
> the classifier never reads all-zero when events exist. `{SCAN_HOURS}` is the
> value Phase 0b.0 resolved (72h after auto-widen).

<!-- VALIDATED 2026-06-03 against live tenant (unguard/unguard-user-simulator,
     72h): reason NULL, signal in event.name; Davis types
     CUSTOM_INFO 5243 / ERROR_EVENT 1549 / WARNING 1117. No dt.system.bucket filter. -->
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize
    oom_killed         = countIf(matchesPhrase(event.name, "OOMKilled") or matchesPhrase(event.name, "OutOfMemory") or matchesPhrase(event.description, "OOMKilled") or reason == "OOMKilled"),
    image_pull_backoff = countIf(matchesPhrase(event.name, "ImagePullBackOff") or matchesPhrase(event.name, "ErrImagePull") or matchesPhrase(event.description, "ImagePullBackOff") or reason == "ImagePullBackOff"),
    crash_loop         = countIf(matchesPhrase(event.name, "CrashLoopBackOff") or matchesPhrase(event.name, "Back-off restarting") or matchesPhrase(event.description, "CrashLoopBackOff") or reason == "CrashLoopBackOff"),
    failed_scheduling  = countIf(matchesPhrase(event.name, "FailedScheduling") or reason == "FailedScheduling"),
    evicted            = countIf(matchesPhrase(event.name, "Evicted") or reason == "Evicted"),
    liveness_failed    = countIf(matchesPhrase(event.name, "Liveness probe failed") or matchesPhrase(event.description, "Liveness probe failed")),
    readiness_failed   = countIf(matchesPhrase(event.name, "Readiness probe failed") or matchesPhrase(event.description, "Readiness probe failed")),
    restarts           = countIf(matchesPhrase(event.name, "Back-off") or matchesPhrase(event.name, "BackOff") or matchesPhrase(event.name, "restarting") or reason == "BackOff" or matchesPhrase(event.description, "restarting")),
    davis_error        = countIf(event.type == "ERROR_EVENT"),
    davis_warning      = countIf(event.type == "WARNING"),
    davis_custom_info  = countIf(event.type == "CUSTOM_INFO")
```

**3) Per-pod restart timeline (15-min bins):**

<!-- VALIDATED 2026-05-28 against live tenant — no dt.system.bucket filter -->
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter matchesPhrase(event.name, "Back-off") or matchesPhrase(event.name, "restarting")
    or matchesPhrase(event.description, "restarting") or reason == "BackOff"
| summarize restarts = count(),
            by: { ts = bin(timestamp, 15m), k8s.pod.name }
| sort ts asc
```

**4) Owner enrichment (only if OWNER_TEAM == null):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch dt.entity.cloud_application
| filter id == "{CLOUD_APP_ID}"
| fieldsAdd metadata.labels, metadata.annotations,
            owner_label = metadata.labels["owner"],
            team_label  = metadata.labels["team"],
            team_anno   = metadata.annotations["dynatrace.com/team"]
```

**PhaseResult shape:**
```json
{
  "event_counts": {
    "oom_killed": 0, "image_pull_backoff": 0, "crash_loop": 0,
    "failed_scheduling": 0, "evicted": 0,
    "liveness_failed": 0, "readiness_failed": 0, "restarts": 0,
    "davis_error": 0, "davis_warning": 0, "davis_custom_info": 0
  },
  "top_events": [
    { "ts": "...", "event.type": "...", "event.name": "...", "reason": "...",
      "pod": "...", "container": "...", "description": "..." }
  ],
  "restart_timeline": [
    { "ts": "...", "pod": "...", "restart_count": 0 }
  ],
  "owner_team": "... or null",
  "event_inflection_time": "ISO or null",
  "window_widened": false,
  "scan_hours_used": 6,
  "gaps": []
}
```

---

#### W-logs — Container Logs Worker

**REFERENCE:** `~/.agents/skills/dt-obs-logs/SKILL.md`.

**dt-k8s-podf scoping** (NOT in the reference — CRITICAL):
- Filter on `timestamp` (NOT `start_time`)
- Scope by NS + WORKLOAD; if POD provided, add pod filter
- Filter to ERROR/WARN: `in(loglevel, array("ERROR", "WARN"))`
- For multi-container workloads, restrict to `CONTAINERS` array from
  DispatchContext to drop istio-proxy access-log noise:
  `| filter in(k8s.container.name, array({CONTAINERS quoted, comma-sep}))`

**Queries to run (ALL in one parallel batch):**

**1) ERROR/WARN logs with JSON parse:**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter in(loglevel, array("ERROR", "WARN"))
| parse content, "JSON:j"
| fields timestamp, loglevel, content,
         msg = j[msg], err = j[err], stack = j[stack],
         k8s.pod.name, k8s.container.name
| sort timestamp desc
| limit 30
```

**2) Exception / timeout / OOM / DNS / connection-refused pattern breakdown:**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter loglevel == "ERROR"
| summarize
    exceptions       = countIf(matchesPhrase(content, "Exception") or matchesPhrase(content, "Traceback")),
    timeouts         = countIf(matchesPhrase(content, "timeout") or matchesPhrase(content, "TimeoutError")),
    oom              = countIf(matchesPhrase(content, "OutOfMemory") or matchesPhrase(content, "OOM")),
    dns_failures     = countIf(matchesPhrase(content, "no such host") or matchesPhrase(content, "DNS")),
    conn_refused     = countIf(matchesPhrase(content, "connection refused") or matchesPhrase(content, "ECONNREFUSED")),
    permission_denied = countIf(matchesPhrase(content, "permission denied") or matchesPhrase(content, "403")),
    panic            = countIf(matchesPhrase(content, "panic") or matchesPhrase(content, "FATAL"))
```

**3) Log error-rate timeseries (5-min bins):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize total = count(),
            errors = countIf(loglevel == "ERROR"),
            warns  = countIf(loglevel == "WARN"),
            by: { ts = bin(timestamp, 5m) }
| fieldsAdd err_pct = (errors * 100.0) / total
| sort ts asc
```

**4) Per-container error distribution (drops sidecar noise):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter loglevel == "ERROR"
| summarize errors = count(), by: { k8s.container.name }
| sort errors desc
```

**PhaseResult shape:**
```json
{
  "top_messages": [
    { "message": "...", "count": 0, "container": "...", "first_seen": "...", "last_seen": "..." }
  ],
  "pattern_counts": {
    "exceptions": 0, "timeouts": 0, "oom": 0,
    "dns_failures": 0, "conn_refused": 0,
    "permission_denied": 0, "panic": 0
  },
  "error_rate_trend": [{ "ts": "...", "err_pct": 0 }],
  "log_inflection_time": "ISO or null",
  "per_container_errors": [{ "container": "...", "errors": 0 }],
  "raw_samples": [{ "timestamp": "...", "loglevel": "...", "content": "..." }],
  "json_parsed": false,
  "gaps": []
}
```

---

#### W-traces — Span Errors Worker

**Skip if `SPAN_PRESENCE == false` in DispatchContext.**

**REFERENCE:** `~/.agents/skills/dt-obs-tracing/references/failure-detection.md`.
Additionally read the protocol-specific reference based on the sample span from
Q5 below — `http-spans.md` if `http.request.method` present, `rpc-spans.md` if
`rpc.system` present, `database-spans.md` if `db.system` present (outbound dep
slowness investigation).

**dt-k8s-podf scoping:** every span query must include
`| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"`
and `start_time` time bounds — NOT a `dt.smartscape.service` filter, which
misses sidecars and pre-instrumentation pods.

**Queries to run (ALL in one parallel batch):**

**1) Failure reason taxonomy** (`failure-detection.md` › Breakdown by Reason):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter request.is_failed == true
| expand dt.failure_detection.results
| summarize fail_count = count(),
            by: { reason = dt.failure_detection.results[reason] }
| sort fail_count desc
```

**2) Exception types** (`failure-detection.md` › Exception-Based Failures):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter iAny(span.events[][span_event.name] == "exception")
| expand span.events
| fieldsFlatten span.events, fields: { exception.type, exception.message }
| summarize exc_count = count(),
            sample_trace = takeAny(trace.id),
            by: { exception.type }
| sort exc_count desc
| limit 10
```

**3) HTTP status / endpoint failure distribution** (skip if no http.* fields):

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter request.is_failed == true
| summarize req_count = count(),
            by: { http.response.status_code, endpoint.name }
| sort req_count desc
| limit 15
```

**4) Outbound dependency latency (server-side view of downstream slowness):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter span.kind == "client"
| summarize call_count = count(),
            avg_ms    = avg(duration) / 1e6,
            p95_ms    = percentile(duration, 95) / 1e6,
            by: { ts = bin(start_time, 15m), span.name }
| sort ts asc
| limit 100
```

**5) Span schema discovery (drives protocol-specific reference selection):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-1h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| limit 1
```

**PhaseResult shape:**
```json
{
  "failure_reasons": { "http_code": 0, "exception": 0, "grpc_code": 0,
                       "span_status": 0, "custom_rule": 0 },
  "top_exceptions": [{ "type": "...", "count": 0, "sample_trace_id": "...", "sample_message": "..." }],
  "failing_paths": [{ "endpoint": "...", "status_code": 0, "count": 0 }],
  "outbound_degradation": [
    { "span_name": "...", "ts": "...", "call_count": 0, "avg_ms": 0, "p95_ms": 0 }
  ],
  "protocols_detected": ["http", "grpc", "db.postgres", ...],
  "span_inflection_time": "ISO or null",
  "gaps": []
}
```

---

#### W-network — NetworkPolicy + Egress Worker

**REFERENCES:**
`~/.agents/skills/dt-obs-kubernetes/references/network-policies.md` and
`labels-annotations.md` (for selector matching).

**Purpose:** surface applied NetworkPolicies and any policy-denial events
("phantom errors" — app code throws DNS/timeout/connection-refused, but the
real cause is a denied egress).

**Queries to run (ALL in one parallel batch):**

**1) Applied NetworkPolicies in the namespace:**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch dt.entity.kubernetes_service
| filter contains(toString(tags), "k8s.namespace.name:{NS}")
| fieldsAdd entity.name, tags, kubernetesAnnotations,
            networkPolicy = kubernetesAnnotations["networking.k8s.io/policy"]
| filter isNotNull(networkPolicy)
| limit 50
```

**2) Pod-egress denial events (if surfaced by CNI integration):**

<!-- VALIDATED 2026-05-28 against live tenant — no dt.system.bucket filter -->
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter matchesPhrase(event.description, "NetworkPolicy")
    or matchesPhrase(event.description, "policy denied")
    or matchesPhrase(event.description, "egress")
    or matchesPhrase(reason, "NetworkPolicy")
| fields ts = timestamp, event.type, reason, event.description, k8s.pod.name
| sort ts desc
| limit 50
```

**3) Outbound destinations attempted (from spans — reveals "what was the pod
trying to reach when denied"):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter span.kind == "client"
| summarize call_count = count(),
            fail_count = countIf(request.is_failed == true),
            by: { server.address, net.peer.name, span.name }
| sort fail_count desc
| limit 20
```

**4) DNS resolution failures (frequent symptom of denied egress to kube-dns):**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch logs, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| filter matchesPhrase(content, "no such host")
    or matchesPhrase(content, "dial tcp: lookup")
    or matchesPhrase(content, "EAI_AGAIN")
    or matchesPhrase(content, "name resolution failed")
| summarize dns_failures = count(),
            sample = takeAny(content),
            by: { k8s.container.name }
| sort dns_failures desc
| limit 10
```

**PhaseResult shape:**
```json
{
  "applied_policies": [{ "name": "...", "selector": "..." }],
  "denial_events": [
    { "ts": "...", "reason": "...", "description": "...", "pod": "..." }
  ],
  "outbound_destinations": [
    { "destination": "...", "span_name": "...", "call_count": 0, "fail_count": 0 }
  ],
  "dns_failures": [
    { "container": "...", "count": 0, "sample_message": "..." }
  ],
  "egress_denied_suspected": false,
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator)

Before trusting any "no events / no logs / no spans / no policies" claim, run
ONE cheap unfiltered confirmation per claim — same rule as dt-rcf RULE 37.

| Worker claim | Confirmation query (drop WORKLOAD filter, keep NS) |
|--------------|----------------------------------------------------|
| "no events" | `fetch events \| filter k8s.namespace.name == "{NS}" \| summarize count(), by: { k8s.workload.name }` |
| "no logs"   | `fetch logs \| filter k8s.namespace.name == "{NS}" \| summarize count(), by: { k8s.workload.name, loglevel }` |
| "no spans"  | `fetch spans \| filter k8s.namespace.name == "{NS}" \| summarize count(), by: { k8s.workload.name }` |
| "no policies" | `fetch dt.entity.kubernetes_service \| filter contains(toString(tags), "k8s.namespace.name:{NS}") \| limit 5` |

If the confirmation query returns rows for OTHER workloads in the same
namespace but zero for `{WORKLOAD}`, that's a genuine workload-scoped absence
(real finding). If it returns zero rows broadly, the namespace itself is
inactive / wrong — escalate as a configuration issue.

Only after a confirmation query genuinely returns zero across the broad scope
may "absent" enter the report.

---

### Phase 2: Cause Classification (Orchestrator, Sequential)

After the four PhaseResults return (and Absence Gate clears any "absent"
claims), classify the root cause into ONE primary class and zero-or-more
contributing classes.

**Cause classes (in order of triage priority):**

| Class | Trigger signals |
|-------|-----------------|
| **resource-oom** | `event_counts.oom_killed > 0` OR `pattern_counts.oom > 0` OR exception type contains "OutOfMemory" |
| **crash-loop** | `event_counts.crash_loop > 0` OR `event_counts.restarts > 0` (broadened PF-K2 counters — these now fire on Davis `CUSTOM_INFO` events whose `event.name` contains `"Back-off restarting failed container …"`, not just raw kubelet `BackOff`) |
| **resource-cpu-throttle** | restart_timeline shows periodic restarts with no app exceptions, AND outbound latency p95 elevated, AND no policy denials |
| **image-pull** | `event_counts.image_pull_backoff > 0` |
| **scheduling** | `event_counts.failed_scheduling > 0` OR `event_counts.evicted > 0` |
| **network-denied-egress** | `egress_denied_suspected == true` OR (`dns_failures > 0` AND `applied_policies` non-empty AND no app exception explains DNS) |
| **network-dns** | `pattern_counts.dns_failures > 0` AND `applied_policies` empty (cluster DNS issue) |
| **probe-config** | `event_counts.liveness_failed + readiness_failed > 0` AND zero app exceptions AND zero failure-reason spans (probe config wrong, app is fine) |
| **app-code** | `top_exceptions[0].count > 0` with trace evidence AND no resource / network signals |
| **dependency** | `outbound_degradation[*].p95_ms` elevated significantly above baseline AND `failure_reasons.http_code > 0` with 5xx |
| **config-drift** | Recent restart correlates with permission_denied / 403 logs |
| **inconclusive** | None of the above signals fire AND the Davis-type counters are also empty (`davis_error + davis_warning + davis_custom_info == 0`) |

> **PF-K2 — never read 0 when events exist.** The classifier MUST inspect the
> broadened `event_counts` (including `davis_error` / `davis_warning` /
> `davis_custom_info`) before falling through to `inconclusive`. If the typed
> kubelet-style counters are all 0 BUT the Davis-type counters are non-zero,
> the events ARE present — pull the actual `event.name` strings from
> `top_events` and classify on their phrase content (e.g. `"Back-off
> restarting"` → `crash-loop`; `"OOMKilled"` → `resource-oom`; probe phrasing →
> `probe-config`). Only declare `inconclusive` when BOTH the kubelet counters
> AND the Davis-type counters are empty across the resolved window.

**Output `CAUSE_CLASSIFICATION`:**
```json
{
  "primary_cause": "<class>",
  "primary_confidence": 0.0-1.0,
  "contributing_causes": ["<class>", ...],
  "evidence_for_primary": [
    { "phase": "events|logs|traces|network", "finding": "..." }
  ],
  "one_sentence_why": "..."
}
```

**Confidence scoring (single primary cause):**
- `1.0` — at least 3 corroborating signals across ≥ 2 phases (e.g., OOM events + OOM log lines + OOM exception type)
- `0.7` — 2 signals in 1 phase (e.g., OOM event count + restart timeline)
- `0.4` — 1 signal only (e.g., one event of the class, no log/trace corroboration)
- `< 0.4` → fall through to `inconclusive` (but see PF-K2 note above — Davis-type
  counters must be empty too)

**PF-K1 auto-widen banner.** If `WINDOW_WIDENED == true` (Phase 0b.0 found 0
events at the default window and retried at 72h), the report Executive Summary
AND the run log MUST carry this banner verbatim — do NOT degrade straight to
`inconclusive` just because the default window was empty:
```
> **Window auto-widened:** the default {DEFAULT_SCAN_HOURS}h window returned 0
> Kubernetes events for this workload, so the investigation was automatically
> re-scoped to {SCAN_HOURS}h, where {EVENT_COUNT_WIDE} events were found
> (types: {PRESENT_EVENT_TYPES}). All phases below reflect the {SCAN_HOURS}h
> window. Recent event rate is low for this workload; if you need the live
> {DEFAULT_SCAN_HOURS}h-only view, re-run with a narrower explicit window.
```

---

### Phase 3: Davis CoPilot Cause-Class Runbook

Pass the **cause classification + evidence** to Davis CoPilot. Do NOT pass raw
phase findings. Ask for a runbook-style remediation specific to the cause
class.

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
KUBERNETES POD DEBUG FORENSICS — WORKLOAD-SCOPED RCA
Namespace:  {NS}
Workload:   {WORKLOAD} ({DEPLOYMENT_KIND})
[ Pod: {POD} ]
Cluster:    {CLUSTER_NAME}
Window:     {SCAN_HOURS}h
Owner:      {OWNER_TEAM}
App version: {APP_VERSION}

CAUSE CLASSIFICATION:
- Primary cause class: {primary_cause} (confidence {primary_confidence})
- Contributing classes: {contributing_causes}
- One-sentence why: {one_sentence_why}

EVIDENCE (cross-phase, ranked):
- [events] {finding}
- [logs]   {finding}
- [traces] {finding}
- [network] {finding}

K8S EVENT COUNTS: {event_counts JSON}
LOG PATTERN COUNTS: {pattern_counts JSON}
FAILURE REASON TAXONOMY: {failure_reasons JSON}
APPLIED NETWORK POLICIES: {applied_policies JSON}

REQUESTED OUTPUT (runbook-style, specific to cause class):
1. Validate or refute the primary cause class given the evidence pattern.
2. Provide concrete remediation steps for the cause class — kubectl commands,
   YAML edits, or runtime-specific tunings (NOT generic "increase resources").
3. Identify the owning team if not already known from labels.
4. Verification step: a single observable that confirms the fix once applied
   (e.g., "OOMKilled events drop to zero", "DNS failures cease", "p95 latency
   returns to baseline").
5. Risk + rollback plan for the remediation.
```

Store the Davis response → drives the report's "Recommended Actions" section.
If Davis is unavailable, derive the runbook from the cause class via the
deterministic table in Appendix C of the report.

---

### Phase 4a: Save Report (Markdown)

**Skip Phase 4a if `NOTEBOOK_MODE == true` and go to Phase 4b.**

**Default: write the `.md` ONLY** — the Markdown is the canonical deliverable.
PDF is generated only when `PDF_MODE == true` (see "Filename + PDF" below) and is
non-fatal.

If `CLEAN_MODE`, build the sanitization map first (identical to dt-rca
Phase 1.14) and compose with sanitized names from the start.

**Emit the report EXACTLY in the section order below — canonical skeleton.**

```markdown
# dt-k8s-podf Forensic Analysis Report
## Pod Debug: {NS} / {WORKLOAD} — {CAUSE_CLASSIFICATION.primary_cause}

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Anchor:** NS={NS}  WORKLOAD={WORKLOAD}  [POD={POD}]
**Cluster:** {CLUSTER_NAME}  |  **Owner:** {OWNER_TEAM}  |  **Workload kind:** {DEPLOYMENT_KIND}
**Window:** last {SCAN_HOURS}h{if WINDOW_WIDENED: " (auto-widened from {DEFAULT_SCAN_HOURS}h — 0 events at default; see banner below)"}  |  **Phases:** events · logs · traces · network

---

## Executive Summary

### Root Cause Class
**{primary_cause}** (confidence {primary_confidence})
{one_sentence_why}

### Impact Summary
| Metric | Value | Status |
|--------|-------|--------|
| Pod restarts ({SCAN_HOURS}h) | {restarts} | … |
| OOMKilled events | {oom_killed} | … |
| Image pull failures | {image_pull_backoff} | … |
| Probe failures (readiness + liveness) | {liveness_failed + readiness_failed} | … |
| Error log volume | {top_messages[0].count} … | … |
| Span failure rate | {failure_rate}% | … |

### Immediate Action
| Priority | Action | Owner | Verification |
|----------|--------|-------|--------------|
[Davis CoPilot runbook output for the primary cause class]

### Restart / Event Timeline
```mermaid
gantt
    title Pod Restart Timeline — {NS}/{WORKLOAD}
    [from restart_timeline — gantt rules per dt-rca]
```

---

## Workload Topology

```
                +-----------------------+
                |  Users / Upstream     |
                +-----------+-----------+
                            |
                +-----------v-----------+
                | Service: {SVC_NAME}   |
                | dt.smartscape.service |
                +-----------+-----------+
                            |
                +-----------v-----------+
                | Workload: {WORKLOAD}  |
                | kind: {DEPLOYMENT_KIND}|
                | containers: {CONTAINERS} |
                | [!!! {primary_cause}] |
                +-----+-------------+---+
                      |             |
              +-------v---+   +-----v-------+
              | Sidecar   |   | Downstream  |
              | (istio…)  |   | (DB / API)  |
              +-----------+   +-------------+
```

[ASCII per dt-rca diagram rules; never raw Mermaid in PDF.]

---

## Event Timeline (Kubernetes)

| Time | Pod | Reason | Description |
|------|-----|--------|-------------|
[From W-events top_events; sort by timestamp; show the 10-15 most diagnostic
 events leading up to the inflection point]

**Restart distribution (per pod):**
[restart_timeline as a small table or sparkline-style row]

---

## Log Excerpts

**Top error/warn patterns:**
| Container | Pattern | Count |
|-----------|---------|-------|
[From W-logs pattern_counts + per_container_errors]

**Sample messages** [sanitized in -clean mode — replace identifiers,
preserve technical content]:
```
[3-5 representative raw_samples — one block per distinct message family]
```

**Log error-rate trend (5-min bins):**
[error_rate_trend as a compact table — log_inflection_time called out]

---

## Span Errors (Distributed Traces)

**Failure taxonomy** (Source: dt-obs-tracing failure-detection.md):
| Failure Reason | Count | % |
|----------------|-------|---|
[failure_reasons breakdown — http_code, exception, grpc_code, span_status, custom_rule]

**Top exception types (span.events):**
| Type | Count | Sample Trace |
|------|-------|--------------|

**Failing endpoints (HTTP):**
| Endpoint | Status | Count |
|----------|--------|-------|

**Outbound dependency degradation:**
| Downstream | call_count | avg ms | p95 ms |
|------------|-----------|--------|--------|

---

## Network Policy Analysis

**Applied NetworkPolicies in namespace `{NS}`:**
| Policy | Selector |
|--------|----------|
[From W-network applied_policies]

**Egress denial signals:**
| Time | Pod | Reason | Description |
|------|-----|--------|-------------|
[From W-network denial_events; if empty, state "no egress denial events surfaced
in window" verbatim]

**Outbound destinations attempted (top 10):**
| Destination | Span Name | Calls | Failures |
|-------------|-----------|-------|----------|

**DNS resolution failures:**
| Container | Count | Sample |
|-----------|-------|--------|

**Egress-denied suspected:** {egress_denied_suspected}

---

## Cross-Phase Evidence Chain

The single authoritative evidence table. Every finding used elsewhere in the
report appears here with its source phase + reference and the cause class it
supports.

| Phase | Source | Finding | Supports |
|-------|--------|---------|----------|
| events | kubernetes/workload-health.md | … | resource-oom |
| logs   | dt-obs-logs | … | resource-oom |
| traces | tracing/failure-detection.md | … | app-code |
| network | kubernetes/network-policies.md | … | network-denied-egress |

---

## Davis CoPilot Synthesis

[Davis CoPilot response from Phase 3 — verbatim sections:
 1. Validation/refutation of primary cause
 2. Concrete remediation steps
 3. Owner identification
 4. Verification observable
 5. Risk + rollback plan]

---

## Recommended Actions

### Immediate (P1)
[Highest-priority Davis action — kubectl/YAML/config — exec prose, NO DQL]

### Follow-Up (P2/P3)
| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
[From Davis + contributing causes]

### Verification
[Single observable that confirms fix — "OOMKilled count drops to 0 in next hour"
 or "p95 latency on downstream X returns to N ms baseline" etc.]

---

## Appendix A: Investigation Details
| Field | Value |
|-------|-------|
| Anchor | NS={NS} WORKLOAD={WORKLOAD} [POD={POD}] |
| Cluster | {CLUSTER_NAME} ({CLUSTER_ID}) |
| Owner Team | {OWNER_TEAM} |
| Workload kind | {DEPLOYMENT_KIND} |
| Containers | {CONTAINERS} |
| Scan window | {SCAN_HOURS}h |
| Phases dispatched | events · logs · traces · network |
| Davis CoPilot | {Used / Unavailable} |
| Primary cause class | {primary_cause} ({primary_confidence}) |
| Contributing causes | {contributing_causes} |

### dt-k8s-podf Telemetry
| Worker | Reference(s) | Key findings | Gaps |
|--------|--------------|--------------|------|
[W-events / W-logs / W-traces / W-network rows]

## Appendix B: Glossary
[Include ONLY if `appendix=true`. Otherwise omit.]

## Appendix C: AI-Powered Analysis Capabilities
[Include ONLY if `appendix=true`. Otherwise omit.]

## Appendix D: Query Exchange Log
[If `appendix=true`: full DQL log with timing for all phases.
 If `appendix=false`: one line → "Full query log: {LOG_PATH}" — do NOT inline DQL.]

---

**Links:**
- [View Workload]({TENANT}/ui/entity/{CLOUD_APP_ID})
- [View Cluster]({TENANT}/ui/entity/{CLUSTER_ID})

---

*End of Report*
```

### Pre-Save Self-Check (run before writing anything)

- [ ] H1 + `## Pod Debug: ...` H2 subtitle present
- [ ] Executive Summary has Root Cause Class + Impact + Action + Timeline
- [ ] Workload Topology box-diagram present
- [ ] Event Timeline shows real K8s events with reasons (not paraphrased)
- [ ] Log Excerpts sanitized in CLEAN_MODE (technical content preserved)
- [ ] Span Errors uses failure-reason taxonomy (NOT just HTTP status)
- [ ] Network Policy section explicitly states applied policies + denial signals
- [ ] Cross-Phase Evidence Chain is the only full evidence table
- [ ] Davis Synthesis verbatim — not paraphrased
- [ ] No DQL anywhere except Appendix D
- [ ] Filename will be `PODF_{NS}_{WORKLOAD}_{DATE}[_SANITIZED]`
- [ ] `.md` is written unconditionally; **PDF only attempted if `PDF_MODE == true`**
      and its absence never fails the run (MD-only is the default expectation)
- [ ] If `WINDOW_WIDENED == true`, the PF-K1 auto-widen banner is present in the
      Executive Summary and the header `Window:` line shows the widened `{SCAN_HOURS}h`
- [ ] Mermaid lint per dt-rcf rules (no `:` in gantt task names; no escaped
      `\"`; `<br/>` for line breaks not `\n`)

### Filename + PDF

**Always write the Markdown file** (canonical deliverable):
```
Normal:   PODF_{NS}_{WORKLOAD}_{DATE}.md
Clean:    PODF_{NS}_{WORKLOAD}_{DATE}_SANITIZED.md
```

**PDF is opt-in and non-fatal.** Only if `PDF_MODE == true`: run the inherited
dt-rca PDF path (two-file strategy — `.md` with Mermaid, `_pdf.md` intermediate
with ASCII, convert via `md-to-pdf`, delete intermediate), producing
`PODF_{NS}_{WORKLOAD}_{DATE}[_SANITIZED].pdf`.

Wrap the conversion so a missing engine NEVER hard-fails the run:
```bash
if command -v md-to-pdf >/dev/null 2>&1; then
  md-to-pdf "PODF_{NS}_{WORKLOAD}_{DATE}_pdf.md" && rm -f "PODF_{NS}_{WORKLOAD}_{DATE}_pdf.md"
else
  echo "PDF engine unavailable — Markdown only"   # one line, then continue
fi
```
If `PDF_MODE == false` (the default), skip all PDF work entirely — do not build
the `_pdf.md` intermediate.

---

### Phase 4b: Notebook Mode (`--notebook`)

**Triggered when `NOTEBOOK_MODE == true`. Replaces Phase 4a entirely.**

Build and deploy a Dynatrace Notebook that mirrors the report sections, with
runnable DQL embedded next to markdown annotations. Reuse the
**dt-pr-notebooks Phase 4** authoring pattern wholesale —
`~/.claude/skills/dt-pr-notebooks/SKILL.md`. Highlights specific to
dt-k8s-podf:

**4b.1 — Validate every DQL before embedding**

For every DQL query you plan to embed, run:
```bash
DTCTL_TOKEN_STORAGE=file dtctl query '<DQL>' --plain
```
If a query fails: fix syntax, retry once. If still failing: include in the
notebook with a markdown note that it requires time-range adjustment. NEVER
embed an unvalidated query.

**4b.2 — Epoch timeframe**

`dt-k8s-podf` defaults to a **relative** time window
(`"defaultTimeframe": { "from": "now()-{SCAN_HOURS}h", "to": "now()" }`) because
pod debug is almost always a live investigation. Use relative `now()-{N}h`
throughout. If the user wants a frozen window, they can pass `--epoch true` and
the skill computes WINDOW_START_MS / WINDOW_END_MS from the inflection
timestamp identified by W-events.

**4b.3 — Notebook section order**

```
1.  [markdown]  Executive Summary
                  - 2-3 sentence summary (cause class, when, owner, impact)
                  - ASCII workload topology
                  - Restart timeline table
                  - "How to use this notebook" step guide

2.  [markdown]  ## Step 1 — Kubernetes Events
                  Explain what the query shows; what restarts/OOM/probe
                  failures look like; what's normal vs incident-grade.

3.  [dql]       W-events Q1 (all workload events)
                  title: "K8s Events — {NS}/{WORKLOAD}"
                  visualization: table

4.  [dql]       W-events Q3 (per-pod restart timeline)
                  title: "Pod Restart Distribution"
                  visualization: areaChart, autoSelectVisualization: false

5.  [markdown]  ## Step 2 — Container Logs (ERROR / WARN)
                  Explain how to read the patterns; what the JSON-parsed
                  fields mean; what a baseline "noisy but healthy" log
                  pattern looks like.

6.  [dql]       W-logs Q1 (ERROR/WARN logs with JSON parse)
                  title: "Container Logs (ERROR/WARN) — last {SCAN_HOURS}h"

7.  [dql]       W-logs Q3 (log error-rate timeseries)
                  title: "Log Error Rate — Inflection at {log_inflection_time}"
                  visualization: lineChart

8.  [markdown]  ## Step 3 — Span Errors (Distributed Traces)
                  Explain failure_reason vs http_status — why span-level
                  ground truth beats HTTP status counts alone.

9.  [dql]       W-traces Q1 (failure reason taxonomy)
                  title: "Failure Reason Taxonomy — {WORKLOAD}"

10. [dql]       W-traces Q2 (exception types from span.events)
                  title: "Exception Types (Top 10) — {WORKLOAD}"

11. [markdown]  ## Step 4 — Network Policy + Egress
                  Explain "phantom errors" pattern — app code throws DNS or
                  timeout, but the real cause is a denied egress NetworkPolicy.
                  How to read the applied-policies table.

12. [dql]       W-network Q1 (applied policies)
                  title: "Applied NetworkPolicies in {NS}"

13. [dql]       W-network Q3 (outbound destinations + failure counts)
                  title: "Outbound Destinations Attempted"

14. [dql]       W-network Q4 (DNS resolution failures)
                  title: "DNS Resolution Failures by Container"

15. [markdown]  ## Step 5 — Root Cause Classification
                  Davis CoPilot synthesis verbatim. Primary cause class +
                  confidence + evidence references.

16. [markdown]  ## Remediation
                  P1/P2 action table.
                  Owner identification.
                  Risk + rollback plan.
                  One sentence: "Run the verification query below — zero
                  records confirms fix applied."
                  NO DQL code blocks in this markdown section.

17. [dql]       Post-fix verification query (cause-class specific)
                  title: "Verification — Zero {failure_signal} Confirms Fix"
                  ⚠ EXCEPTION: always from:now()-15m (live state — verifies fix
                  is currently holding)
                  See cause-class table below for the verification query template.
```

**Cause-class verification query templates** (used in Section 17):

| primary_cause | Verification query (returns 0 when fixed) |
|---------------|--------------------------------------------|
| resource-oom | `fetch events, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter reason == "OOMKilled" or matchesPhrase(event.description, "OOMKilled")` |
| image-pull | `fetch events, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter reason == "ImagePullBackOff" or matchesPhrase(event.description, "ImagePullBackOff")` |
| scheduling | `fetch events, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter reason == "FailedScheduling" or reason == "Evicted"` |
| network-denied-egress | `fetch logs, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter matchesPhrase(content, "no such host") or matchesPhrase(content, "connection refused")` |
| probe-config | `fetch events, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter matchesPhrase(event.description, "Liveness probe failed") or matchesPhrase(event.description, "Readiness probe failed")` |
| app-code | `fetch spans, from:now()-15m \| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}" \| filter iAny(span.events[][span_event.name] == "exception") \| filter iAny(span.events[][exception.type] == "{top_exception_type}")` |

**4b.4 — Deploy**

```bash
bash ~/.claude/skills/dt-app-notebooks/scripts/deploy_notebook.sh /tmp/{NS}-{WORKLOAD}-podf.json
```

The deploy script validates all DQL queries and blocks on failures. On success
it prints the notebook URL — relay that to the user via Phase 7 output.

---

### Phase 5: Output Summary (Phase 7 in run log)

```markdown
## dt-k8s-podf Investigation Complete

| Artifact | Filename / URL |
|----------|----------------|
[Default mode:]
| Markdown | PODF_{NS}_{WORKLOAD}_{DATE}.md |
| PDF | {if PDF_MODE: PODF_{NS}_{WORKLOAD}_{DATE}.pdf — else: `skipped (MD-only default; pass --pdf to enable)`} |
| Log | PODF_{NS}_{WORKLOAD}_{DATE}.log |

[--notebook mode:]
| Notebook | {NOTEBOOK_URL} |
| Log | PODF_{NS}_{WORKLOAD}_{DATE}.log |

**Anchor:** NS={NS}  WORKLOAD={WORKLOAD}  [POD={POD}]
**Cluster:** {CLUSTER_NAME}
**Owner:** {OWNER_TEAM}
**Window:** {SCAN_HOURS}h

### Cause Classification
- **Primary:** {primary_cause} ({primary_confidence})
- **Contributing:** {contributing_causes}
- **Why:** {one_sentence_why}

### Worker Findings
| Worker | Reference | Headline | Gaps |
|--------|-----------|----------|------|
| W-events | kubernetes/workload-health + pod-debugging | {restarts} restarts, {oom_killed} OOMKilled | … |
| W-logs   | dt-obs-logs | {top_messages[0].count}× "{top_messages[0].message}" | … |
| W-traces | tracing/failure-detection | {top_failure_reason}: {N} spans | … |
| W-network | kubernetes/network-policies | {applied_policies count} policies, egress denied: {bool} | … |

### Immediate Action
{Davis CoPilot P1 action — one line}

### Verification
Run the verification query in {Notebook Section 17 / Report "Verification"
section} — zero records confirms the fix is applied.
```

If CLEAN_MODE, append the sanitization key to the console (never to file) per
dt-rca rules.

---

## RULES (NON-NEGOTIABLE)

### Inherited from dt-rca / dt-rcf

**Rules 1–16 from dt-rca** (`~/.claude/skills/dt-rca/SKILL.md` › RULES) apply
unchanged: no skipped data collection; Mermaid-in-MD / ASCII-in-PDF; executive
audience; Appendix-D query logging; compact diagrams; two-file PDF strategy;
clean-mode zero-leaks / consistency / plausibility; DQL backtick / timestamp /
count-alias hygiene; canonical link format; service-ID-for-spans.

**Rules 28, 29b, 36, 37 from dt-rcf** apply unchanged: parallel worker dispatch
in single message; absence-claim proof + Absence Gate; dtctl auth on-disk /
lazy-login / orchestrator-only; error/empty ≠ absence. Login is
interactive-and-orchestrator-only; workers never log in or refresh, and the
orchestrator never races concurrent/non-interactive logins (see Auth Hardening
2026-06-03).

### New in dt-k8s-podf

1. **K8s anchor is the canonical scope.** Every events/logs/spans query MUST
   include `| filter k8s.namespace.name == "{NS}"` AND
   `| filter k8s.workload.name == "{WORKLOAD}"`, plus the optional
   `| filter k8s.pod.name == "{POD}"` when POD is provided. NEVER scope by
   `dt.entity.service` alone — it misses sidecars, init containers, and
   pods that haven't attached to a registered service entity.

2. **Logs on `timestamp`, spans on `start_time`, events on `timestamp`
   (no `dt.system.bucket` filter — tenant-validated 2026-05-28).** This is the
   most common DQL pitfall — wrong time field returns zero rows and looks like
   absence. The legacy `dt.system.bucket == "default_events"` constraint also
   returns zero rows on this tenant family; rely on `k8s.namespace.name` +
   `k8s.workload.name` for scope instead.

3. **Every `bin(...)` must be ALIASED.** Use `ts = bin(timestamp, 5m) | sort ts
   asc` — never `sort \`bin(timestamp, 5m)\`` (backticks break in
   `dtctl query '...'` and `start_time`/`timestamp` no longer exist post-summarize).

4. **Container filter is required for multi-container workloads.** If
   `CONTAINERS` has more than one entry, every log query must restrict to the
   primary app container (or the explicit container list) — otherwise
   istio-proxy access logs drown out the signal. The reference for the rule
   is `labels-annotations.md`.

5. **Cause classification is mandatory.** Phase 2 must produce a typed
   `primary_cause` from the classes in the table — never publish the report
   with a free-form prose root cause. `inconclusive` is the only allowed
   non-class value.

6. **Davis runbook is cause-class specific.** Phase 3 must pass the cause
   class label (not raw findings) to Davis CoPilot. Davis is expected to
   produce concrete remediation (`kubectl set resources …`, NetworkPolicy
   YAML, probe tuning) — generic advice ("scale up", "check logs") fails the
   pre-save self-check.

7. **Verification observable is required.** The report's "Verification"
   subsection — and the notebook's Section 17 DQL — must define a single
   observable that returns 0 / drops to baseline once the fix is applied.
   No verification = report rejected. The cause-class verification template
   table in Phase 4b is the canonical source.

8. **NetworkPolicy phantom-error check is mandatory.** W-network runs on every
   investigation regardless of cause hypothesis. Many "app exceptions" are
   really denied egress; skipping this worker silently mislabels the cause
   as `app-code`.

9. **Filename = `PODF_{NS}_{WORKLOAD}_{DATE}[_SANITIZED]`** — never `Problem_*`,
   never `RCF_*`.

10. **`--notebook` and report mode are mutually exclusive.** A single invocation
    emits one artifact set, not both. The argument parser sets `NOTEBOOK_MODE`
    once and Phase 4 branches to exactly one of 4a (Markdown report; PDF only if
    `PDF_MODE`) / 4b (notebook). `--pdf` is a sub-flag of report mode and is
    ignored when `NOTEBOOK_MODE == true`.

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"PDF DIAGRAM RULES". Summary: no Mermaid in PDFs — ASCII art only, inside
plain code blocks, 80-char max width.

---

## MERMAID SYNTAX RULES

Inherits dt-rca's "MERMAID SYNTAX RULES" PLUS the dt-rcf LABEL-SANITIZATION
rules (no colon in gantt task names; no literal double-quotes in sequence
messages; `<br/>` not `\n` for line breaks; no raw `file:line` — use
`file Lnn`; no ISO timestamps with seconds in labels — `HH:MM` only).

dt-k8s-podf-specific addition: **pod names contain hyphens and hex suffixes
(`checkout-api-7d4f9-xkj2p`); use them verbatim in tables but shorten to the
workload + `…-xkj2p` form in diagram labels** — the full pod name overflows
80-char ASCII boxes and breaks Mermaid label rendering.

---

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"DIAGRAM SIZING RULES". Summary: `graph LR` for cascades > 4 nodes; 3-4 words
per node; no deeply nested subgraphs.

---

## ERROR HANDLING

### No workload found for the anchor
```markdown
## Error: Workload Not Found

Anchor `NS:"{NS}" WORKLOAD:"{WORKLOAD}"` returned zero results across events,
logs, and spans in the last {SCAN_HOURS}h.

**Possible causes:**
- Namespace or workload name spelled differently in this tenant
- Workload deleted more than {SCAN_HOURS}h ago (try widening with manual
  Phase 0b query, or re-run after confirming the names in the Dynatrace UI)
- Tenant mismatch — confirm `dtctl auth whoami` shows the expected environment

**Try:** List workloads in the namespace via
`fetch dt.entity.cloud_application | filter contains(toString(tags),
"k8s.namespace.name:{NS}") | fields entity.name | limit 20`
```

### A worker fails entirely
Orchestrator marks the corresponding section with `gap`, notes the worker name
+ reference + failure mode in Appendix A, and continues. Does not auto-retry
after the single re-dispatch from Phase 1.

### Davis CoPilot unavailable
Generate the report (or notebook) from the cause classification alone. Add the
banner:
```
> **Note:** Davis CoPilot synthesis unavailable. Remediation guidance derived
> from cause class + deterministic runbook table (Appendix C). Verify the
> verification observable before considering the workload healthy.
```

### Cause classification == `inconclusive`
Only reachable when BOTH the kubelet-style counters AND the Davis-type counters
(`davis_error` / `davis_warning` / `davis_custom_info`) are empty across the
resolved window (PF-K2). If `WINDOW_WIDENED == true`, the auto-widen to 72h has
already been attempted — say so in the banner so the reader knows widening is not
the next step. Report findings as-is but add the banner:
```
> **Inconclusive:** Forensic evidence collected but no high-confidence cause
> class identified{if WINDOW_WIDENED: " — even after auto-widening to {SCAN_HOURS}h"}.
> Review the Cross-Phase Evidence Chain and consider:
> (1) {if WINDOW_WIDENED: "widening manually beyond {SCAN_HOURS}h" else: "widening the scan window beyond {SCAN_HOURS}h"},
> (2) targeting a specific pod with POD:"...",
> (3) running /dt-rcf SERVICE:"<service-name>" for a service-centric view.
```

---

## SKILLS USED BY THIS SKILL

| Skill | When used |
|-------|-----------|
| `dt-obs-kubernetes` | W-events (workload-health, pod-debugging, labels-annotations, pod-node-placement), W-network (network-policies, ingress) |
| `dt-obs-logs` | W-logs (search, filter, pattern analysis, error-rate timeseries) |
| `dt-obs-tracing` | W-traces (failure-detection, http/rpc/database spans based on detected protocol) |
| `dt-dql-essentials` | DQL syntax + smartscapeNodes / getNodeName signatures (Phase 0c) |
| `dt-pr-notebooks` | Phase 4b notebook JSON authoring + deploy_notebook.sh pattern (only when `--notebook`) |
| `dt-rca` | Phase 1.14 sanitization map; PDF generation; Mermaid + ASCII rules |
| `dt-rcf` | Worker dispatch protocol, Absence Gate, dtctl auth hygiene |

**Do NOT use** `dt-rca` or `dt-rcf` as outputs — dt-k8s-podf is the terminal
skill for this domain. It composes their patterns but never delegates to them.

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously until
the report (or notebook) is saved/deployed.

**Do not stop for confirmation between phases. Do not ask questions. Generate
the complete artifact.**
