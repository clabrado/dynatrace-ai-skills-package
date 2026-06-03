---
name: dt-deploy-risk
description: >-
  Deployment Risk Scorecard for Dynatrace. Given a DEPLOY:<event-id> anchor (or
  SERVICE:"<name>" VERSION:"<v>" fallback), measures the delta between a
  configurable pre-deploy baseline window and the equivalent post-deploy window
  across RED metrics, exception types, log patterns, and downstream dependency
  health. Composes dt-obs-services, dt-obs-tracing, dt-obs-problems, and
  dt-obs-logs in a parallel worker dispatch (one message, four agents), scores
  the deploy 0-100 via a weighted rubric (error 0.35 / latency 0.25 / new
  exceptions 0.25 / dep churn 0.15), assigns a GO / HOLD / ROLLBACK verdict,
  passes the scorecard to Davis CoPilot for recommendation synthesis, and
  emits a Markdown report (PDF optional via `--pdf`). Use after a deployment
  when you need a defensible go/rollback decision in minutes — not after the
  next page fires. Requires deployment-event ingestion (SDLC_EVENT
  `task.deployment.finished`, or legacy CUSTOM_DEPLOYMENT / DEPLOYMENT_EVENT);
  fails fast if no matching event is found in a 48h window around the
  SERVICE+VERSION fallback.
---

# dt-deploy-risk — Dynatrace Deployment Risk Scorecard

Post-deployment delta analyzer. Compares a configurable BASELINE window
(default 24h before T0) against the matching POST window (T0 → T0 + baseline,
or `now()` if the deploy is too recent), scores the deploy on a weighted
rubric, and emits a defensible GO / HOLD / ROLLBACK verdict with evidence.

This skill is NOT an incident-investigation skill. It is a **delta comparator**:
it always asks "what changed between BEFORE and AFTER?" — never "what's broken
right now?". For raw incident triage use `/dt-rcf` or `/dt-rca`.

---

## Usage

```
/dt-deploy-risk DEPLOY:abc123                                       # Deploy event ID anchor (preferred)
/dt-deploy-risk SERVICE:"checkout-api" VERSION:"v2.41.0"            # Service + version fallback
/dt-deploy-risk DEPLOY:abc123 --baseline 7d                         # 7-day baseline window
/dt-deploy-risk SERVICE:"payments" VERSION:"build-7714" --baseline 4h
/dt-deploy-risk DEPLOY:abc123 -clean                                # Sanitized report
/dt-deploy-risk DEPLOY:abc123 --pdf                                 # Also render PDF (MD is the default deliverable)
```

## Arguments

- `$ARGUMENTS` — exactly one anchor (DEPLOY OR SERVICE+VERSION). Both forms
  must resolve to a single deployment event; ambiguous matches fail Phase 0b.
- `--baseline <duration>` — comparison window length on each side of T0.
  Accepts `4h`, `24h` (default), `3d`, `7d`. The POST window uses the same
  duration unless capped at `now()`.
- `-clean` — sanitize all identifying names per dt-rca Phase 1.14 rules.
- `appendix=true` — include Appendix B (Glossary) + C (Capabilities); default OFF.

> **Speed note:** run at **medium effort**. High effort multiplies subagent
> latency without improving the scorecard. Quality is set by reference reads
> and baseline-window correctness, not effort level.

---

## Sub-Skills Loaded Per Phase

dt-deploy-risk carries NO comparison DQL. Each worker reads ONE targeted
reference file at start (once per subagent), derives query patterns from it,
and applies the BEFORE-vs-AFTER scoping below. References stay current
upstream; this skill stays correct automatically.

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-red          | `~/.claude/skills/dt-obs-services/references/service-metrics.md` + runtime ref (java.md / nodejs.md / etc., auto-detected) |
| W-exceptions   | `~/.claude/skills/dt-obs-tracing/references/failure-detection.md` |
| W-logs         | `~/.claude/skills/dt-obs-logs/SKILL.md` |
| W-deps         | `~/.claude/skills/dt-obs-tracing/references/failure-detection.md` (outbound client-span patterns) |

---

## Skill Registry

Maintenance-time reference map for inline DQL provenance. Used when
re-validating; NOT read during normal runs. **Never guess a path, metric, or
field — fix it here and re-validate against the tenant.**

**Re-validation gate:** every inline query in this skill MUST be run via
`dtctl query '<dql>'` before saving when EDITED. Common breakers: backticks
(use aliased bins); `timestamp` vs `start_time`; `dt.entity.service` (does
not exist); ambiguous `dt.service.name` (prefer span-derived); silent
empty rows from a filter on the frequently-empty `dt.process_group.id`.

### DQL Authority

| File | Purpose | When to read |
|------|---------|--------------|
| `~/.claude/skills/dtctl/references/DQL-reference.md` | Core DQL syntax | Orchestrator (Phase 0c) |
| `~/.claude/skills/dt-dql-essentials/references/dql/dql-functions-smartscape.md` | smartscapeNodes / getNodeName signatures | Orchestrator (Phase 0c) |

### W-red — dt-obs-services

| File | When to read |
|------|--------------|
| `~/.claude/skills/dt-obs-services/SKILL.md` | Always |
| `~/.claude/skills/dt-obs-services/references/service-metrics.md` | Always — RED metric patterns |
| `~/.claude/skills/dt-obs-services/references/java.md` | RUNTIME == java OR unknown |
| `~/.claude/skills/dt-obs-services/references/nodejs.md` | RUNTIME == nodejs |
| `~/.claude/skills/dt-obs-services/references/dotnet.md` | RUNTIME == dotnet |
| `~/.claude/skills/dt-obs-services/references/python.md` | RUNTIME == python |
| `~/.claude/skills/dt-obs-services/references/php.md` | RUNTIME == php |
| `~/.claude/skills/dt-obs-services/references/go.md` | RUNTIME == go |

### W-exceptions — dt-obs-tracing

| File | When to read |
|------|--------------|
| `~/.claude/skills/dt-obs-tracing/SKILL.md` | Always |
| `~/.claude/skills/dt-obs-tracing/references/failure-detection.md` | Always — exception + failure reason patterns |

### W-logs — dt-obs-logs

| File | When to read |
|------|--------------|
| `~/.claude/skills/dt-obs-logs/SKILL.md` | Always |

### W-deps — dt-obs-tracing

| File | When to read |
|------|--------------|
| `~/.claude/skills/dt-obs-tracing/references/failure-detection.md` | Always — outbound client-span latency + failure patterns |

---

**Sanitization, URL format, Mermaid/ASCII rules, and PDF generation** are
identical to `/dt-rca` — read `~/.claude/skills/dt-rca/SKILL.md` sections
"Phase 1.14", "Phase 3", "Phase 4", "PDF DIAGRAM RULES", "MERMAID SYNTAX RULES",
"DIAGRAM SIZING RULES". All apply unchanged.

---

## ENTITY MODEL — SMARTSCAPE-NATIVE (NON-NEGOTIABLE)

dt-deploy-risk uses the Smartscape entity model throughout. `dt.entity.*`
filters on spans/logs/events are prohibited.

### Span / Log / Event Filtering

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
-- Filter spans to the target service
fetch spans
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE
    | filter id == "{SERVICE_ENTITY_ID}"
    | fields id
]
```

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
-- Filter logs to the target service (logs filter on timestamp, NOT start_time)
fetch logs
| filter dt.smartscape_source.id in [
    smartscapeNodes SERVICE
    | filter id == "{SERVICE_ENTITY_ID}"
    | fields id
]
| filter timestamp >= toTimestamp("{from}") AND timestamp <= toTimestamp("{to}")
```

<!-- VALIDATED 2026-06-03 via dtctl query (5 deploys returned for Easy Travel 0.1.1) -->
```dql
-- Deploy events: SDLC_EVENT carries NO entity link — match on component.* fields,
-- NOT on dt.smartscape_source.id (which is empty on SDLC_EVENT). The legacy
-- CUSTOM_DEPLOYMENT/DEPLOYMENT_EVENT path is kept only as a probe candidate (see
-- Phase 0b.0). On this tenant SDLC_EVENT is the live path.
fetch events
| filter event.kind == "{DEPLOY_EVENT_KIND}" and event.type == "{DEPLOY_EVENT_TYPE}"
| filter component.name == "{SERVICE_NAME}" and component.version == "{VERSION}"
```
> `{DEPLOY_EVENT_KIND}` / `{DEPLOY_EVENT_TYPE}` are bound by the Phase 0b.0
> Substrate Probe (primary `SDLC_EVENT` / `task.deployment.finished`). The
> legacy `CUSTOM_DEPLOYMENT` path matched on `dt.smartscape_source.id` only
> applies if the probe selects that candidate.

### Process Group Scope for Runtime Metrics

Runtime / process metrics are scoped by `dt.entity.process_group_instance`
(a standard metric dimension). NEVER filter on `dt.process_group.id` — it is
frequently EMPTY and yields false zero-row results.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
timeseries jvm_heap = avg(dt.runtime.jvm.memory.heap.used),
           by: {dt.smartscape.process, dt.entity.process_group_instance}
| filter dt.entity.process_group_instance == "{ROOT_PGI}"
```

### Entity Name Resolution

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans
| fieldsAdd service_name = getNodeName(dt.smartscape.service),
            downstream_name = getNodeName(dt.smartscape.downstream_service)
```

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking)

Set `LOG="DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}.log"` next to the report. At each
phase boundary append ONE line, backgrounded:

```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X - <summary>" >> "$LOG"; } &
```

Log: START, Phase 0b.0 Substrate Probe one-liner (selected deploy
event.kind/event.type + version field + service-link/bridge status), Phase 0b
complete (DispatchContext summary with T0 + windows + bridged ROOT_SERVICE_ID),
Phase 1 complete (per-worker one-liners), Absence-Gate result, scoring
breakdown, verdict, report written, PDF status (only if PDF_MODE),
RUN COMPLETE. Phase 7 prints the log path.

---

### Phase 0a: Anchor Parser

```
ANCHOR_TYPE + ANCHOR_VALUE:
  DEPLOY:<id>                                        -> DEPLOY_EVENT
  SERVICE:"<name>" VERSION:"<v>"                     -> SERVICE_VERSION
  (no anchor, both anchors, conflicting forms)       -> ERROR + usage

FLAGS:
  --baseline <duration>     -> BASELINE_DURATION (default "24h"); accepts 4h, 24h, 3d, 7d
  -clean                    -> CLEAN_MODE = true
  appendix=true             -> FULL_APPENDIX = true (default false)
  --pdf                     -> PDF_MODE = true (default FALSE; MD is the canonical deliverable)
  pdf=true                  -> alias for --pdf (PDF_MODE = true)
  pdf=false                 -> no-op (MD-only is already the default)

VALIDATION: if no anchor parseable, or both forms supplied, print usage and exit.
```

Parse BASELINE_DURATION into a duration object. The same value is reused for
the POST window (capped at `now()`).

---

### Phase 0-auth: PRE-WARM the dtctl session (Orchestrator)

FIRST action of the whole run — before any data query, before any dispatch:

```
dtctl auth refresh
```

This is a REFRESH, not a login — silent, no browser, uses the on-disk refresh
token (`DTCTL_TOKEN_STORAGE=file`, harness env, never the Keychain). It
guarantees a fresh access token up front so workers all inherit one token
instead of racing concurrent refreshes. Refresh is consistent with "never
proactively log in" — refresh is not login.

- `dtctl auth refresh` succeeds -> proceed to Phase 0b.
- It fails (no refresh token / weeks idle) -> ONLY THEN run once:
  `dtctl auth login --plain --safety-level readonly` (read-only scopes; one-time browser).

Workers NEVER authenticate. On `<gap: auth>`, the orchestrator refreshes once
and re-dispatches that ONE worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

---

### Phase 0b: Context Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** that all phase workers receive. Bootstrap is
the orchestrator's only serial stretch — keep it tight.

#### Phase 0b.0 — Substrate Probe (run ONCE, before the hardcoded bootstrap)

Deploy-event substrate drifts (the 2026-05-28 snapshot claimed CUSTOM_DEPLOYMENT;
the live tenant emits SDLC_EVENT). Do NOT assume — **discover the shape at
runtime** and bind it into variables the later queries interpolate.

**Probe (a) — which deploy `event.kind` / `event.type` exists:**

<!-- VALIDATED 2026-06-03 via dtctl query -->
```dql
fetch events, from:now()-30d
| filter event.type == "task.deployment.finished"
       or event.kind == "SDLC_EVENT"
       or event.kind == "CUSTOM_DEPLOYMENT"
       or event.kind == "DEPLOYMENT_EVENT"
       or event.type == "CUSTOM_DEPLOYMENT"
| summarize c = count(), by: { event.kind, event.type }
| sort c desc
| limit 20
```

Select from the **CANDIDATE LIST** (ordered `[corrected-2026-06-03, legacy]`),
binding the FIRST candidate that returns a non-zero count:

| Rank | DEPLOY_EVENT_KIND | DEPLOY_EVENT_TYPE | Version field(s) | Service link |
|---|---|---|---|---|
| 1 (primary) | `SDLC_EVENT` | `task.deployment.finished` | `component.version` (component name = `component.name`) | none — needs the component→service bridge below |
| 2 (legacy) | `DAVIS_EVENT` (or `CUSTOM_DEPLOYMENT`) | `CUSTOM_DEPLOYMENT` | `event.version_to` / `event.deployment.version` / `properties.version` | `dt.smartscape_source.id` |
| 3 (legacy) | `DEPLOYMENT_EVENT` | `DEPLOYMENT_EVENT` | `event.version_to` / `event.deployment.version` | `dt.smartscape_source.id` |

> Probe note (validated 2026-06-03): on this tenant the legacy
> `event.type == "CUSTOM_DEPLOYMENT"` rows arrive under
> `event.kind == "DAVIS_EVENT"`, NOT the assumed `DEPLOYMENT_EVENT`. Bind the
> candidate by what the probe ACTUALLY returns — match on `event.type` for the
> legacy CUSTOM_DEPLOYMENT path rather than hardcoding its `event.kind`.

Bind `DEPLOY_EVENT_KIND`, `DEPLOY_EVENT_TYPE`, `VERSION_FIELDS`, and
`EVENT_HAS_SERVICE_LINK` (true only for legacy candidates 2/3). The PRIMARY
candidate (SDLC_EVENT / task.deployment.finished) is the validated live path
(5,106 events / 30d); pick it whenever it returns rows.

Emit a one-line probe summary to the run log, e.g.
`Probe: deploy event.kind→SDLC_EVENT, event.type→task.deployment.finished, version field→component.version, service link→NONE (bridge required)`.

**If NONE of the candidates return rows**, abort with
`## Error: Deploy Event Not Found` — absence is gated, never assumed.

#### Resolve the deploy event

##### If ANCHOR_TYPE == DEPLOY_EVENT

<!-- VALIDATED 2026-06-03 via dtctl query (SDLC_EVENT path) -->
```dql
fetch events, from:now()-30d
| filter event.kind == "{DEPLOY_EVENT_KIND}" and event.type == "{DEPLOY_EVENT_TYPE}"
| filter event.id == "{ANCHOR_VALUE}"
| fieldsAdd ts = timestamp
| sort ts desc
| limit 1
```

Extract: `T0` (timestamp), `SERVICE_NAME` (`component.name` for the SDLC_EVENT
candidate; for legacy candidates use `dt.smartscape_source.id` directly as
`ROOT_SERVICE_ID`), `VERSION` (from the probe's `VERSION_FIELDS`),
`DEPLOY_NAME` (event.name or component.name), `DEPLOY_AUTHOR`
(deployment.team / properties.author, may be null).

When `EVENT_HAS_SERVICE_LINK == false` (SDLC_EVENT), set `ROOT_SERVICE_ID` via
the **component→service bridge** below.

##### If ANCHOR_TYPE == SERVICE_VERSION

Step 1 — find the deploy event by component/version match (SDLC_EVENT primary):

<!-- VALIDATED 2026-06-03 via dtctl query (5 deploys for Easy Travel 0.1.1) -->
```dql
fetch events, from:now()-30d
| filter event.kind == "{DEPLOY_EVENT_KIND}" and event.type == "{DEPLOY_EVENT_TYPE}"
| filter component.name == "{SERVICE_NAME}" and component.version == "{VERSION}"
| fieldsAdd ts = timestamp
| sort ts desc
| limit 5
```

> Legacy candidates (2/3) have no `component.*` fields — for those, scope by
> `dt.smartscape_source.id in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]`
> and match `contains(event.version_to, "{VERSION}")`. Resolve `ROOT_SERVICE_ID`
> first via the component→service bridge below (which doubles as name→entity
> resolution for legacy too).

**Fail-fast rule:** if zero rows abort with `## Error: Deploy Event Not Found`.
If >1 distinct `event.id` exist for the same `component.name`/`component.version`,
that is NORMAL for SDLC_EVENT (deploys recur — the validated tenant emits a new
event every ~10 min). Take the MOST RECENT (`sort ts desc | limit 1`) as T0;
do NOT abort on multiplicity for the SDLC_EVENT candidate. (The legacy
candidates retain the strict ">1 distinct event.id within 48h → abort" rule,
since a single CUSTOM_DEPLOYMENT version should be unique.)

Extract: `T0`, `DEPLOY_EVENT_ID`, `VERSION`, `DEPLOY_NAME`.

#### Component → Service bridge (DR-4 — required for SDLC_EVENT)

The SDLC_EVENT carries **no `dt.entity.service` / `dt.smartscape_source.id`
link** (validated 2026-06-03 — the only fields are `component.name`,
`component.version`, `deployment.stage`, openpipeline metadata). The four delta
workers MUST scope to a real `SERVICE-*` id, so bridge `component.name` to a
Smartscape SERVICE entity. **Dynatrace MCP is NOT registered on this tenant —
`dtctl` is the PRIMARY path; MCP is a fallback only.**

Step A — dtctl Smartscape bridge (PRIMARY). Case- and token-tolerant, because
`component.name` ("Easy Travel") rarely equals the service display name
("easyTravel Customer Frontend", "JourneyService"):

<!-- VALIDATED 2026-06-03 via dtctl query (resolved "Easy Travel" → multiple easyTravel services) -->
```dql
smartscapeNodes SERVICE
| fieldsAdd lname = lower(name)
| filter contains(lname, lower("{SERVICE_NAME}"))
      or contains(lname, replaceString(lower("{SERVICE_NAME}"), " ", ""))
| fields id, name
| limit 25
```

> Note: `component.name` "Easy Travel" did NOT exact-match any SERVICE on the
> validated tenant — service names use the camel/no-space form
> ("easyTravel …") and per-component names ("JourneyService"). The
> space-stripped, lower-cased `contains` is what catches "easyTravel …".

Step B — if Step A returns exactly ONE service, bind it as `ROOT_SERVICE_ID`.
If it returns MULTIPLE, the component maps to a service GROUP (common for a
deployable "component" spanning several services). Pick the highest-traffic
member as `ROOT_SERVICE_ID` by ranking candidates over the POST window. (T0 is
already known here, so the POST window is computed in "Compute windows" below;
run this ranking step after that — or rank over a `from:now()-{BASELINE_DURATION}`
window inline if you want to bind `ROOT_SERVICE_ID` before window computation.)

<!-- VALIDATED 2026-06-03 via dtctl query (ranks all easyTravel services by traffic) -->
```dql
fetch spans, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE
    | fieldsAdd lname = lower(name)
    | filter contains(lname, lower("{SERVICE_NAME}"))
          or contains(lname, replaceString(lower("{SERVICE_NAME}"), " ", ""))
    | fields id
  ]
| summarize spancount = count(), by: { dt.smartscape.service }
| fieldsAdd service_name = getNodeName(dt.smartscape.service)
| sort spancount desc
| limit 10
```
> DQL note: a smartscape-id filter MUST use the `smartscapeNodes … | fields id`
> subquery form (as above). A bare `in ["SERVICE-…", …]` string-literal array
> is REJECTED (`PARSE_ERROR`); `in(dt.smartscape.service, array("SERVICE-…"))`
> works but emits a `toSmartscapeId()` warning — prefer the subquery.

Bind the top row as `ROOT_SERVICE_ID`; record the other candidates in
`SERVICE_BRIDGE_CANDIDATES` for Appendix A so the operator can see the mapping
and re-anchor on a specific service if the auto-pick is wrong. (Validated:
"Easy Travel" → top `SERVICE-…` EasyTravelBackendWebserver:8091 / 4.4M spans,
with `easyTravel Customer Frontend`, `EasytravelService`, etc. as siblings.)

Step C — MCP fallback (ONLY if dtctl Step A returns zero rows AND the MCP is
registered): `mcp__dynatrace__find_entity_by_name` with `SERVICE_NAME`. If the
MCP is unavailable AND Step A is empty, abort with
`## Error: Service Entity Not Resolved` (see Error Handling) — never score
against an unresolved service.

Extract: `ROOT_SERVICE_ID` (SERVICE-XXXX), `ROOT_ENTITY_NAME` (the bridged
service display name), `SERVICE_BRIDGE_CANDIDATES`.

#### Compute windows

```
BASELINE_WINDOW = { from: T0 - BASELINE_DURATION, to: T0 }
POST_WINDOW     = { from: T0,                     to: min(T0 + BASELINE_DURATION, now()) }
POST_IS_PARTIAL = (T0 + BASELINE_DURATION) > now()
POST_COVERAGE_PCT = floor( (now() - T0) / BASELINE_DURATION * 100 )  if POST_IS_PARTIAL else 100
```

If `POST_COVERAGE_PCT < 25`, abort with `## Error: Insufficient Post-Deploy Window`
(coverage below the noise floor; user should wait or shrink `--baseline`).

#### Resolve runtime + PGI + k8s (ONE combined query)

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE
    | filter id == "{ROOT_SERVICE_ID}"
    | fields id
  ]
| summarize spancount = count(),
            by: { dt.entity.process_group_instance, k8s.namespace.name, k8s.workload.name }
| sort spancount desc
| limit 1
```

Set `ROOT_PGI` (a `PROCESS_GROUP_INSTANCE-…` ID, NOT a `PROCESS_GROUP-…` ID),
`K8S_NAMESPACE`, `K8S_WORKLOAD`. If the POST window has too few spans (e.g.
deploy 5 minutes ago, no traffic yet), re-run against BASELINE_WINDOW.

RUNTIME is usually evident from the PGI detected name (e.g. "index.js
checkout-*" -> nodejs). If still unknown, run the legacy metadata query:

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch dt.entity.process_group
| filter id == "{PROCESS_GROUP_ID}"
| fieldsAdd softwareTechnologies, metadata
```

#### Store as DispatchContext JSON

```json
{
  "ANCHOR_TYPE": "DEPLOY_EVENT|SERVICE_VERSION",
  "ANCHOR_VALUE": "...",
  "DEPLOY_EVENT_KIND": "SDLC_EVENT",
  "DEPLOY_EVENT_TYPE": "task.deployment.finished",
  "EVENT_HAS_SERVICE_LINK": false,
  "DEPLOY_EVENT_ID": "...",
  "T0": "ISO",
  "VERSION": "...",
  "DEPLOY_NAME": "...",
  "DEPLOY_AUTHOR": "... or null",
  "ROOT_SERVICE_ID": "SERVICE-XXXX",
  "ROOT_ENTITY_NAME": "...",
  "SERVICE_BRIDGE_CANDIDATES": ["SERVICE-XXXX (name, spancount)", "..."],
  "ROOT_PGI": "PROCESS_GROUP_INSTANCE-XXXX or null",
  "K8S_NAMESPACE": "... or null",
  "K8S_WORKLOAD": "... or null",
  "RUNTIME": "java|nodejs|dotnet|python|php|go|unknown",
  "BASELINE_DURATION": "24h",
  "BASELINE_WINDOW": { "from": "ISO", "to": "ISO" },
  "POST_WINDOW":     { "from": "ISO", "to": "ISO" },
  "POST_IS_PARTIAL": false,
  "POST_COVERAGE_PCT": 100,
  "CLEAN_MODE": false,
  "FULL_APPENDIX": false
}
```

---

### Phase 0c: Reference Strategy

dt-deploy-risk is **reference-driven** — workers carry no hardcoded
comparison DQL. Each worker reads ONE targeted reference file at start, once
per subagent, derives the BEFORE / AFTER query pair from it, and applies the
delta-comparison scoping below.

Orchestrator exceptions (inline only): the Phase 0b.0 Substrate Probe, the
Phase 0b bootstrap (deploy resolver + component→service bridge), and Phase 1.5
Absence-Gate confirmation queries. Everything else lives in the workers.

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

**Dispatch ALL FOUR workers in ONE message with multiple Agent calls — true
concurrency.** Sequential dispatch (send, wait, send) defeats the architecture
and is a top cause of slow runs.

Workers W-red, W-exceptions, W-logs, W-deps run concurrently as subagents
(`subagent_type: Explore`). Each worker MUST execute its queries TWICE — once
against BASELINE_WINDOW, once against POST_WINDOW — then compute the delta
internally and return only the delta (never raw rows from both windows).

**Speed model:** each worker reads ONE targeted reference file (5-20 KB,
once), derives queries, fires them in parallel. Phase 0b has pre-warmed the
dtctl token. Expected total: bootstrap + worker fan-out + synthesis ≈ 6-8 min
at medium effort.

**Worker-failure handling — NO silent serial fallback.** If a worker returns
gaps or fails, do NOT inline its queries in the orchestrator thread. Instead:
re-dispatch that ONE worker ONCE (fresh Agent call with a note on what to
fix). If it still fails, record a `gap` for that component and apply the
penalty rule in Phase 2 (gap penalty = 50 — neither best nor worst).

#### Worker Prompt Template

```
You are a deployment-risk worker for dt-deploy-risk.

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 - READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed in "REFERENCE FILES" below. Read each file
ONCE. Take query patterns from the reference; apply the DISPATCH CONTEXT
scoping (Smartscape entity IDs, window timestamps, dtctl hygiene rules).
This is the ONLY source of DQL truth - do NOT derive queries from training
knowledge or guess field names.

STEP 2 - RUN ALL QUERIES IN ONE PARALLEL BATCH (BOTH windows simultaneously):
For each metric, issue the BASELINE query and the POST query AT THE SAME TIME
in one message (multiple dtctl query tool calls). Do not read files and query
at the same time - read first, then query.

ENTITY MODEL - CRITICAL:
- NEVER use dt.entity.service/host == "..." to filter spans, logs, or events.
  Use: filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "ID" | fields id]
- NEVER use classicEntitySelector().
- Process/runtime metrics: scope by dt.entity.process_group_instance == "{ROOT_PGI}"
  (a standard metric dimension). dt.process_group.id is frequently EMPTY - NEVER filter on it.
- For entity names: getNodeName(dt.smartscape.service) / getNodeName(dt.smartscape.host)
- LOG queries filter on `timestamp`. SPAN queries filter on `start_time`.

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

QUERIES TO RUN:
{QUERY_INSTRUCTIONS_FOR_THIS_WORKER}

RULES:
- EXPLORATION CAP: run ONLY the queries defined in "QUERIES TO RUN". No
  ad-hoc "let me also check..." queries.
- PREFLIGHT: run `DTCTL_TOKEN_STORAGE=file dtctl query 'fetch spans | limit 1'`.
  If it errors with an auth/connection problem (not a DQL error), return
  immediately with <gap: dtctl-unavailable: {message}> and STOP.
- Run BASELINE and POST queries for each metric in parallel (one message,
  multiple tool calls). Compute deltas LOCALLY. Return ONLY the delta.
- Do NOT return raw rows. Internalize; summarize aggressively.
- Cap your PhaseResult at ~4 KB. Top-5 of anything, not top-50.
- If a query fails with a SYNTAX/FIELD error, retry ONCE with corrected syntax
  (check: aliased bin not backticked; `timestamp` for logs vs `start_time` for
  spans; single-quoted dtctl arg).
- AUTH IS THE ORCHESTRATOR'S JOB - workers NEVER run dtctl auth login OR
  dtctl auth refresh. If dtctl returns an auth error AND every call has
  DTCTL_TOKEN_STORAGE=file, return <gap: auth> and STOP - never attempt any
  auth recovery.
- ERROR != ABSENCE. A query that ERRORED tells you nothing about whether
  data exists. Only a SUCCESSFUL query returning zero rows is evidence of
  absence - and if that query was scoped, CONFIRM with a broader query before
  concluding "no data".
- PROOF OF ABSENCE (required): if you DO conclude "no exceptions / no errors /
  no traffic" for the POST window, include the unfiltered confirmation query
  you ran and its zero-row result.
- Execute DQL via `DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'` - prefix
  EVERY dtctl call. SINGLE QUOTES around the DQL.
- NO BACKTICKS in DQL. ALIAS every bin: `by:{ ts = bin(start_time, 5m) } | sort ts asc`.

RETURN only this PhaseResult shape (JSON). Nothing else.
{PHASE_RESULT_SCHEMA}
```

---

#### W-red — RED metrics delta (rate, errors, duration)

**REFERENCE-DRIVEN — dt-deploy-risk carries NO RED DQL; dt-obs-services is
the authority.** Read `~/.claude/skills/dt-obs-services/references/service-metrics.md`
ONCE plus the runtime file for the detected RUNTIME
(`java.md` / `nodejs.md` / `dotnet.md` / `python.md` / `php.md` / `go.md`).
Take the patterns; apply the delta-comparison scoping below.

**dt-deploy-risk MUST-KEEPS (overrides for OTel services):**
- Derive RED from root spans, not `dt.service.request.*` metrics
  (`dt.entity.service` does not exist; `dt.service.name` is ambiguous for
  OTel and yields false "no metrics" findings).
- For runtime metrics, scope by `dt.entity.process_group_instance == "{ROOT_PGI}"`.
- If a window has no spans, do NOT report it as "service down" — verify with
  an unfiltered confirmation query before concluding.

**Queries to run (ALL in one parallel batch — BASELINE + POST for each):**

1. **RED summary (span-derived)** — once for BASELINE, once for POST:
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
   | filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
   | filter request.is_root_span == true
   | filter start_time >= toTimestamp("{from}") AND start_time <= toTimestamp("{to}")
   | summarize reqs   = count(),
               fails  = countIf(request.is_failed == true),
               p50_ms = percentile(duration, 50) / 1e6,
               p95_ms = percentile(duration, 95) / 1e6,
               p99_ms = percentile(duration, 99) / 1e6
   | fieldsAdd err_pct = (fails * 100.0) / reqs
   ```

2. **RED per-interval trend (POST only)** — 5-min bins for the burn chart:
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
   | filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
   | filter request.is_root_span == true
   | summarize reqs = count(),
               fails = countIf(request.is_failed == true),
               p95_ms = percentile(duration, 95) / 1e6,
               by: { ts = bin(start_time, 5m) }
   | fieldsAdd err_pct = (fails * 100.0) / reqs
   | sort ts asc
   ```

3. **Runtime metrics delta** — from the runtime reference. dt-deploy-risk
   MUST-KEEPS:
   - For **nodejs**: validated metric names are `dt.runtime.nodejs.eventloop.utilization`
     and `dt.runtime.nodejs.memory.used` — NOT `dt.runtime.nodejs.v8heap.*`.
   - For **java**: `dt.runtime.jvm.memory.heap.used` and
     `dt.runtime.jvm.gc.collection_count`.
   - Scope by `dt.entity.process_group_instance == "{ROOT_PGI}"`.
   - VERIFY BEFORE CONCLUDING ABSENCE: if 0 rows, re-run grouped by process
     with no PGI filter. Other processes returning data means the scope was
     wrong, not "not instrumented".

**PhaseResult shape (delta-only):**
```json
{
  "reqs_baseline": 0,
  "reqs_post": 0,
  "reqs_delta_pct": 0.0,
  "err_pct_baseline": 0.0,
  "err_pct_post": 0.0,
  "err_pct_delta_abs": 0.0,
  "p50_ms_baseline": 0,
  "p50_ms_post": 0,
  "p50_ms_delta_pct": 0.0,
  "p95_ms_baseline": 0,
  "p95_ms_post": 0,
  "p95_ms_delta_pct": 0.0,
  "p99_ms_baseline": 0,
  "p99_ms_post": 0,
  "p99_ms_delta_pct": 0.0,
  "post_trend": [{ "ts": "ISO", "reqs": 0, "err_pct": 0.0, "p95_ms": 0 }],
  "runtime": "java|nodejs|...",
  "runtime_signals": [{ "metric": "...", "baseline": 0, "post": 0, "delta_pct": 0.0 }],
  "gaps": []
}
```

---

#### W-exceptions — new exception types (set-difference)

**REFERENCE-DRIVEN — read `~/.claude/skills/dt-obs-tracing/references/failure-detection.md`
ONCE.** Take the exception-extraction patterns; apply the delta-comparison
scoping below.

**dt-deploy-risk MUST-KEEPS:**
- Spans filter on `start_time` (NOT `timestamp`).
- Build the BASELINE exception-type set and the POST exception-type set
  separately, then return the set-difference (POST minus BASELINE) plus
  occurrence counts in POST.
- An exception present in BOTH windows is NOT a regression — it is
  pre-existing. Only NEW exception types penalize the score.

**Queries to run (ALL in one parallel batch — BASELINE + POST):**

1. **Exception types (run twice, one window each):**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
   | filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
   | filter start_time >= toTimestamp("{from}") AND start_time <= toTimestamp("{to}")
   | filter iAny(span.events[][span_event.name] == "exception")
   | expand span.events
   | fieldsFlatten span.events, fields:{ exception.type }
   | summarize count    = count(),
               sample   = takeAny(trace.id)
               by: { exception.type }
   | sort count desc
   | limit 50
   ```

2. **Failure reason breakdown (POST only)** — context for the new exceptions:
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
   | filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
   | filter request.is_failed == true
   | expand dt.failure_detection.results
   | summarize count = count(),
               by: { reason = dt.failure_detection.results[reason] }
   | sort count desc
   ```

**Delta computation (in-worker):**
```
BASELINE_TYPES = set of exception.type from query 1 against BASELINE_WINDOW
POST_TYPES     = set of exception.type from query 1 against POST_WINDOW
NEW_TYPES      = POST_TYPES - BASELINE_TYPES
For each type in NEW_TYPES: report count_in_post + sample_trace_id.
```

**PhaseResult shape:**
```json
{
  "new_exception_types": [
    { "type": "java.sql.SQLTransientConnectionException",
      "post_count": 0,
      "sample_trace_id": "..." }
  ],
  "exception_types_persistent": [{ "type": "...", "baseline_count": 0, "post_count": 0 }],
  "failure_reasons_post": { "http_code": 0, "exception": 0, "grpc_code": 0, "span_status": 0, "custom_rule": 0 },
  "gaps": []
}
```

---

#### W-logs — new error / warn log patterns (set-difference)

**REFERENCE-DRIVEN — read `~/.claude/skills/dt-obs-logs/SKILL.md` ONCE**
(sections: "Log Searching", "Log Filtering", "Pattern Analysis").

**dt-deploy-risk MUST-KEEPS (critical — not in the reference):**
- Logs filter on **`timestamp`**, NOT `start_time`.
- Scope with the 3-way OR — Smartscape PLUS k8s fallback:
  ```
  filter dt.smartscape_source.id in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
      or dt.source_entity     in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
      or (k8s.namespace.name == "{K8S_NAMESPACE}" and k8s.container.name == "{K8S_WORKLOAD}")
  ```
  The k8s clause is REQUIRED — the Smartscape source filter returns 0 for
  OTel services.
- Pattern matching: use `takeAny(content)` and the first 80 chars of `content`
  as a pattern key (`pattern_key = substring(content, 0, 80)`). Group by
  `pattern_key` to fold near-duplicates without doing expensive regex parsing.

**Queries to run (ALL in one parallel batch — BASELINE + POST):**

1. **Error/warn log patterns (run twice, one window each):**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch logs, from:toTimestamp("{from}"), to:toTimestamp("{to}")
   | filter (dt.smartscape_source.id in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id])
         or (dt.source_entity     in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id])
         or (k8s.namespace.name == "{K8S_NAMESPACE}" and k8s.container.name == "{K8S_WORKLOAD}")
   | filter timestamp >= toTimestamp("{from}") AND timestamp <= toTimestamp("{to}")
   | filter in(loglevel, array("ERROR", "WARN"))
   | fieldsAdd pattern_key = substring(content, 0, 80)
   | summarize count   = count(),
               sample  = takeAny(content),
               by: { pattern_key, loglevel }
   | sort count desc
   | limit 100
   ```

2. **Log error-rate burn (POST only):**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch logs, from:toTimestamp("{POST_WINDOW.from}"), to:toTimestamp("{POST_WINDOW.to}")
   | filter (dt.smartscape_source.id in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id])
         or (k8s.namespace.name == "{K8S_NAMESPACE}" and k8s.container.name == "{K8S_WORKLOAD}")
   | summarize total  = count(),
               errors = countIf(loglevel == "ERROR"),
               by: { ts = bin(timestamp, 5m) }
   | fieldsAdd err_pct = (errors * 100.0) / total
   | sort ts asc
   ```

**Delta computation (in-worker):**
```
BASELINE_PATTERNS = set of pattern_key from BASELINE
POST_PATTERNS     = set of pattern_key from POST
NEW_PATTERNS      = POST_PATTERNS - BASELINE_PATTERNS
For each pattern in NEW_PATTERNS: report count_in_post + sample_content (truncated to 200 chars).
```

**PhaseResult shape:**
```json
{
  "new_error_patterns": [
    { "pattern_key": "ERROR Database connection refused...",
      "loglevel": "ERROR",
      "post_count": 0,
      "sample": "..." }
  ],
  "patterns_persistent": [{ "pattern_key": "...", "baseline_count": 0, "post_count": 0 }],
  "log_error_rate_trend": [{ "ts": "ISO", "err_pct": 0.0 }],
  "gaps": []
}
```

---

#### W-deps — downstream dependency health delta

**REFERENCE-DRIVEN — read `~/.claude/skills/dt-obs-tracing/references/failure-detection.md`
ONCE.** Take the outbound client-span patterns; apply delta scoping.

**Queries to run (ALL in one parallel batch — BASELINE + POST):**

1. **Outbound dependency latency (run twice, one window each):**
   <!-- VALIDATE: run via dtctl query against tenant before first production use -->
   ```dql
   fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
   | filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{ROOT_SERVICE_ID}" | fields id]
   | filter start_time >= toTimestamp("{from}") AND start_time <= toTimestamp("{to}")
   | filter span.kind == "client"
   | fieldsAdd downstream_name = coalesce(getNodeName(dt.smartscape.downstream_service),
                                          server.address,
                                          "unknown")
   | summarize calls       = count(),
               fails       = countIf(request.is_failed == true),
               avg_ms      = avg(duration) / 1e6,
               p95_ms      = percentile(duration, 95) / 1e6,
               by: { downstream_name }
   | fieldsAdd err_pct = (fails * 100.0) / calls
   | sort calls desc
   | limit 25
   ```

**Delta computation (in-worker):**
```
For each downstream service in POST:
  - If absent in BASELINE: NEW dependency (churn).
  - If avg_ms_post > avg_ms_baseline * 1.25: latency_regression.
  - If err_pct_post > err_pct_baseline + 1.0: error_regression.
For each downstream service present in BASELINE but missing in POST: REMOVED dependency (churn).
Return only services that changed (NEW / REMOVED / regression).
```

**PhaseResult shape:**
```json
{
  "new_dependencies": [{ "downstream_name": "...", "post_calls": 0, "post_err_pct": 0.0 }],
  "removed_dependencies": [{ "downstream_name": "...", "baseline_calls": 0 }],
  "latency_regressions": [
    { "downstream_name": "...",
      "avg_ms_baseline": 0,
      "avg_ms_post": 0,
      "delta_pct": 0.0 }
  ],
  "error_regressions": [
    { "downstream_name": "...",
      "err_pct_baseline": 0.0,
      "err_pct_post": 0.0,
      "delta_abs": 0.0 }
  ],
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator — run before trusting any "no change" or "no data" finding)

Before scoring, scan every PhaseResult for absence claims ("no new exceptions",
"no new log patterns", "0 requests", "no runtime data", "no downstream calls").
For EACH such claim:

1. Require the worker's **proof** (unfiltered confirmation query + its
   zero-row result). If missing, the claim is invalid.
2. Run ONE cheap unfiltered confirmation query yourself. Examples:
   - **No POST traffic:** `fetch spans, from:POST_WINDOW | filter dt.smartscape.service in [...root...] | summarize count()` — if zero, the service may genuinely have no traffic OR the deploy broke ingestion. Flag for the verdict.
   - **No new exceptions but spike in failures:** confirm `request.is_failed == true` count in POST is also zero. If non-zero, W-exceptions missed something — re-dispatch.
   - **No new logs:** confirm with the k8s.namespace + container fallback.
3. Only after confirmation may "no change" enter the report.

**A worker's empty result is a hypothesis to verify, never a conclusion to score.**

---

### Phase 2: Scorecard Computation (Orchestrator, Sequential)

Compute the four penalties from the worker PhaseResults. Each penalty is
0-100 (100 = worst). Cap inputs to prevent any one signal from saturating
the final score.

#### Penalty rubric

```
error_delta_penalty (W-red):
  d = err_pct_post - err_pct_baseline    (absolute percentage-point delta)
  if d <= 0:        penalty = 0
  elif d <= 0.5:    penalty = 20         (within noise floor)
  elif d <= 2.0:    penalty = 50
  elif d <= 5.0:    penalty = 75
  else:             penalty = 100

latency_delta_penalty (W-red):
  Use the WORSE of p95_delta_pct and p99_delta_pct.
  d = max(p95_delta_pct, p99_delta_pct)
  if d <= 0:        penalty = 0
  elif d <= 10:     penalty = 20
  elif d <= 25:     penalty = 50
  elif d <= 50:     penalty = 75
  else:             penalty = 100

new_exception_penalty (W-exceptions):
  n = count of new_exception_types
  total_count = sum of post_count across new types
  if n == 0:        penalty = 0
  elif n == 1 and total_count < 10:  penalty = 30
  elif n <= 2:      penalty = 60
  elif n <= 5:      penalty = 85
  else:             penalty = 100

dep_churn_penalty (W-deps):
  churn = len(new_dependencies) + len(removed_dependencies)
        + len(latency_regressions) + len(error_regressions)
  if churn == 0:    penalty = 0
  elif churn == 1:  penalty = 30
  elif churn <= 3:  penalty = 60
  elif churn <= 5:  penalty = 85
  else:             penalty = 100

Gap rule: if a worker returned <gap>, set its penalty = 50 (neither best
nor worst) and flag it in the report header.
```

#### Score formula

```
score = 100 - (error_delta_penalty * 0.35)
            - (latency_delta_penalty * 0.25)
            - (new_exception_penalty * 0.25)
            - (dep_churn_penalty * 0.15)

# round to nearest integer; clamp [0, 100]
```

#### Verdict

```
score >= 80  -> GO        (deploy is safe; continue rollout)
50 <= score < 80 -> HOLD  (manual review required; do not auto-promote)
score < 50   -> ROLLBACK  (revert recommended)
```

#### POST_IS_PARTIAL adjustment

If `POST_IS_PARTIAL == true` and `POST_COVERAGE_PCT < 50`, downgrade the
verdict by one tier (GO -> HOLD, HOLD -> ROLLBACK is NOT applied — never
auto-recommend rollback from partial data). Add banner:
```
> Partial post-deploy window: {POST_COVERAGE_PCT}% of {BASELINE_DURATION}
> elapsed since T0. Verdict downgraded one tier. Re-run after full coverage.
```

---

### Phase 3: Davis CoPilot Synthesis (Orchestrator, Sequential)

Pass the **scorecard** to Davis CoPilot — NOT the raw PhaseResults.

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
DEPLOYMENT RISK SCORECARD: {ROOT_ENTITY_NAME} version {VERSION}
DEPLOY EVENT: {DEPLOY_EVENT_ID} at {T0}
BASELINE WINDOW: {BASELINE_WINDOW.from} to {BASELINE_WINDOW.to}
POST WINDOW: {POST_WINDOW.from} to {POST_WINDOW.to} (coverage: {POST_COVERAGE_PCT}%)

SCORE: {score}/100        VERDICT: {GO|HOLD|ROLLBACK}

PENALTY BREAKDOWN:
  Error rate delta   ({weight} 35%): {error_delta_penalty}/100
    err_pct {baseline} -> {post} (delta {delta_abs}pp)
  Latency delta      ({weight} 25%): {latency_delta_penalty}/100
    p95 {baseline}ms -> {post}ms ({delta_pct}%), p99 {baseline}ms -> {post}ms ({delta_pct}%)
  New exceptions     ({weight} 25%): {new_exception_penalty}/100
    {n} new types, top: {type} ({count}x)
  Dependency churn   ({weight} 15%): {dep_churn_penalty}/100
    {n_new} new deps, {n_removed} removed, {n_regress} regressions

KEY SIGNALS:
  - {top new exception type with sample trace}
  - {top new log pattern with sample message}
  - {largest dependency regression}

QUESTIONS:
1. Validate the verdict ({GO|HOLD|ROLLBACK}) against the evidence. Does the
   penalty pattern match a known deployment failure mode (config drift,
   schema mismatch, dependency timeout, memory leak, OOM, GC pressure)?
2. For the recommended action ({deploy proceed | hotfix specific area |
   rollback}), what are the highest-leverage next steps?
3. What additional signal (specific metric, specific log query) would
   increase confidence in the verdict? Be concrete.
```

Davis CoPilot response feeds the "Recommended Action" section. If
unavailable, use the rubric-derived recommendation alone with the banner from
ERROR HANDLING below.

---

### Phase 4: Report Generation

If CLEAN_MODE, build the sanitization map first (identical to dt-rca
Phase 1.14) and compose with sanitized names from the start.

**Emit the report EXACTLY in the section order below — this is the canonical
skeleton, not a menu. Each fact has ONE canonical home; never restate the
same data in another section.**

```markdown
# dt-deploy-risk Scorecard
## Deployment {VERSION} of {ROOT_ENTITY_NAME} — Verdict: {GO|HOLD|ROLLBACK}

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Deploy Event:** {DEPLOY_EVENT_ID}  |  **T0:** {T0}  |  **Baseline:** {BASELINE_DURATION}
**Windows:** baseline [{BASELINE_WINDOW.from} -> {BASELINE_WINDOW.to}], post [{POST_WINDOW.from} -> {POST_WINDOW.to}]
**Post coverage:** {POST_COVERAGE_PCT}%   {partial-window banner if applicable}

---

## Executive Verdict

> **{VERDICT}** — score **{score}/100**. {one-sentence rationale from Davis CoPilot, or rubric-derived if Davis unavailable.}

### Recommended Action
| Priority | Action | Owner | Urgency |
|----------|--------|-------|---------|
[From Davis CoPilot recommendation, or rubric-derived. Concrete: "Rollback to {prior version}", "Hotfix DB pool config", "Continue rollout".]

---

## Score Breakdown

| Component | Weight | Penalty (0-100) | Weighted | Notes |
|-----------|--------|-----------------|----------|-------|
| Error rate delta | 35% | {n} | {n*0.35} | err_pct {b}% -> {p}% (Δ {d}pp) |
| Latency delta    | 25% | {n} | {n*0.25} | p95 {b}ms -> {p}ms ({d}%); p99 {b}ms -> {p}ms ({d}%) |
| New exceptions   | 25% | {n} | {n*0.25} | {count} new types, total {n} occurrences |
| Dependency churn | 15% | {n} | {n*0.15} | {new}/{removed}/{regress} new/removed/regressed |
| **Total**        |     |     | **{score}** | Verdict threshold: GO ≥80, HOLD 50–79, ROLLBACK <50 |

---

## Before / After Metrics

### RED Summary
| Metric | Baseline | Post | Delta |
|--------|----------|------|-------|
| Requests           | {n}    | {n}    | {pct}%  |
| Error rate         | {pct}% | {pct}% | {pp}pp  |
| p50 latency        | {ms}   | {ms}   | {pct}%  |
| p95 latency        | {ms}   | {ms}   | {pct}%  |
| p99 latency        | {ms}   | {ms}   | {pct}%  |

### Post-Deploy Burn (5-min bins)
```mermaid
xychart-beta
    title "Error rate post-deploy"
    x-axis [t0, t1, t2, ...]
    y-axis "Error %" 0 --> 100
    line [...]
```
[Generated from W-red post_trend. If too sparse, omit the chart and present a 5-row max table.]

### Runtime Metrics ({RUNTIME})
| Metric | Baseline | Post | Delta |
|--------|----------|------|-------|
[From W-red runtime_signals — e.g. JVM heap, GC count, Node event-loop utilization]

---

## New Exceptions (Post Only)

| Exception Type | Post Count | Sample Trace |
|----------------|-----------|--------------|
[From W-exceptions.new_exception_types — set-difference, POST minus BASELINE. If empty, write: "No new exception types introduced by this deployment."]

**Failure reason distribution (post):** http_code {n}, exception {n}, grpc_code {n}, span_status {n}, custom_rule {n}.

---

## New Log Patterns (Post Only)

| Pattern (first 80 chars) | Level | Post Count | Sample |
|--------------------------|-------|-----------|--------|
[From W-logs.new_error_patterns. Sample truncated to ≤200 chars. If empty: "No new ERROR/WARN log patterns introduced."]

---

## Dependency Impact

### Latency Regressions
| Downstream | Baseline avg | Post avg | Delta |
|------------|--------------|----------|-------|
[From W-deps.latency_regressions, >25% threshold]

### Error Regressions
| Downstream | Baseline err% | Post err% | Delta |
|------------|---------------|-----------|-------|
[From W-deps.error_regressions]

### Dependency Churn
| Change | Service | Detail |
|--------|---------|--------|
[NEW / REMOVED dependencies from W-deps]

---

## Davis CoPilot Synthesis

[Full Davis response: verdict validation, failure-mode classification,
recommended next steps. If unavailable, omit this section and add the
"Davis unavailable" banner under Executive Verdict.]

---

## Appendix A: Investigation Details

| Field | Value |
|-------|-------|
| Anchor type | {DEPLOY_EVENT \| SERVICE_VERSION} |
| Deploy event kind/type | {DEPLOY_EVENT_KIND} / {DEPLOY_EVENT_TYPE} (probe-selected) |
| Deploy event ID | {DEPLOY_EVENT_ID} |
| Deploy component | {SERVICE_NAME} v{VERSION} ({deployment.stage if present}) |
| Service entity (bridged) | {ROOT_SERVICE_ID} ({ROOT_ENTITY_NAME}) |
| Bridge candidates | {SERVICE_BRIDGE_CANDIDATES — "single exact match" if only one} |
| Runtime | {RUNTIME} |
| Root PGI | {ROOT_PGI or "not resolved"} |
| K8s | namespace={K8S_NAMESPACE} workload={K8S_WORKLOAD} |
| Baseline duration | {BASELINE_DURATION} |
| Post coverage | {POST_COVERAGE_PCT}% |
| Workers dispatched | W-red, W-exceptions, W-logs, W-deps |
| Davis CoPilot | {Used \| Unavailable} |

### dt-deploy-risk Telemetry
| Worker | Sub-Skill | Key Delta | Gaps |
|--------|-----------|-----------|------|
| W-red        | dt-obs-services/{runtime}.md | ... | ... |
| W-exceptions | dt-obs-tracing/failure-detection.md | ... | ... |
| W-logs       | dt-obs-logs | ... | ... |
| W-deps       | dt-obs-tracing/failure-detection.md | ... | ... |

## Appendix B: Glossary
[Include ONLY if FULL_APPENDIX == true.]

## Appendix C: AI-Powered Analysis Capabilities
[Include ONLY if FULL_APPENDIX == true.]

## Appendix D: Query Exchange Log
[If FULL_APPENDIX == true: full DQL log with timing for all phases.
 Otherwise: "Full query log: {LOG_PATH}".]

---

**Links:**
- [View Deploy Event](https://{TENANT}.apps.dynatrace.com/ui/apps/dynatrace.classic.events/event/{DEPLOY_EVENT_ID})
- [View Service](https://{TENANT}.apps.dynatrace.com/ui/entity/{ROOT_SERVICE_ID})

---

*End of Report*
```

### Pre-Save Self-Check (run before Phase 5 writes anything)

The canonical deliverable is **Markdown** — these checks all target the `.md`.
PDF is opt-in (`--pdf`) and is verified only when `PDF_MODE == true`; a missing
PDF engine is never a self-check failure.

If any item is "no", fix before writing:
- [ ] H1 + `## Deployment {VERSION} of {SERVICE} — Verdict: {V}` subtitle
- [ ] Header has Generated / Analyst / Environment / Deploy / T0 / Baseline /
      Windows / Post coverage
- [ ] Executive Verdict has the one-sentence verdict callout and the
      Recommended Action table
- [ ] Score Breakdown table has all four components + weights + Total row
- [ ] Before/After metrics table present (RED summary)
- [ ] Runtime table present matching detected RUNTIME (or "not collected" row)
- [ ] New Exceptions section present (empty or populated)
- [ ] New Log Patterns section present (empty or populated)
- [ ] Dependency Impact has three subsections (latency / error / churn) —
      empty subsections explicitly state "No change"
- [ ] Davis section present OR Davis-unavailable banner under Verdict
- [ ] Appendix A present; B + C only if FULL_APPENDIX; D present (full or pointer)
- [ ] Filename `DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}[_SANITIZED]`
- [ ] No DQL anywhere outside Appendix D
- [ ] Footer is `**Links:**` + `*End of Report*`
- [ ] Mermaid lint — same rules as dt-rcf (no `file:line` in labels;
      sequence text after `:` has no `"` and no second `:`; node labels use
      `<br/>` not `\n`)

---

### Phase 5: Save Report (Markdown)

Run the Pre-Save Self-Check, then **write the `.md` ONLY — Markdown is the
canonical deliverable.** PDF is opt-in and must NEVER block or fail a run.

Filenames:
```
Normal: DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}.md
Clean:  DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}_SANITIZED.md

ANCHOR_SLUG examples:
  DEPLOY:abc123                                      -> DEPLOY_abc123
  SERVICE:checkout-api VERSION:v2.41.0               -> SERVICE_checkout_api_v2_41_0
```

**PDF (ONLY if `PDF_MODE == true`):** attempt the inherited dt-rca PDF path —
read `~/.claude/skills/dt-rca/SKILL.md` Phases 3+4 for the `md-to-pdf` command,
the intermediate `_pdf.md` strategy (Mermaid -> ASCII conversion), and cleanup.
**Wrap it so a missing engine is non-fatal:** if `md-to-pdf` (or the configured
engine) is unavailable, append ONE line to the run log
(`PDF engine unavailable — Markdown only`) and continue. Never hard-fail the
run on PDF. PDF filenames mirror the MD names with a `.pdf` extension.

---

### Phase 6: Output Summary

```markdown
## dt-deploy-risk Scorecard Generated

| Format | Filename |
|--------|----------|
| Markdown | DEPLOY_RISK_{SLUG}.md |
| PDF | {DEPLOY_RISK_{SLUG}.pdf if PDF_MODE else `skipped (MD-only default; pass --pdf to enable)`} |
| Log | DEPLOY_RISK_{SLUG}.log |

**Deploy:** {DEPLOY_EVENT_ID} — {ROOT_ENTITY_NAME} {VERSION}
**T0:** {T0}
**Windows:** baseline [{B.from} -> {B.to}], post [{P.from} -> {P.to}] ({POST_COVERAGE_PCT}% coverage)

### Verdict: {GO | HOLD | ROLLBACK}
**Score:** {score}/100

| Component | Weight | Penalty | Weighted |
|-----------|--------|---------|----------|
| Error rate delta | 35% | {n} | {w} |
| Latency delta    | 25% | {n} | {w} |
| New exceptions   | 25% | {n} | {w} |
| Dependency churn | 15% | {n} | {w} |

### Key Signals
- **Error rate:** {b}% -> {p}% (Δ {d}pp)
- **p95 latency:** {b}ms -> {p}ms ({d}%)
- **New exceptions:** {n} type(s); top: {type} ({count}x)
- **Dependency churn:** {new} new, {removed} removed, {regress} regressed

### Recommended Action
{One-line action from Davis CoPilot, or rubric-derived}
```

If CLEAN_MODE, append sanitization key to console (never to file) per dt-rca rules.

---

## RULES (NON-NEGOTIABLE)

### Inherited from dt-rca

**Rules 1-16 are identical to dt-rca's RULES — apply unchanged. See
`~/.claude/skills/dt-rca/SKILL.md` "RULES (NON-NEGOTIABLE)".** They cover:
no skipped data collection; Mermaid-in-MD / ASCII-in-PDF; executive
audience; Appendix-D query logging; historical verification; compact
diagrams; two-file PDF strategy; clean-mode zero-leaks; DQL backtick /
timestamp / count-alias rules; canonical-link and service-ID-for-spans rules.

### New in dt-deploy-risk — comparison engine

17. **Smartscape-native entity model.** NEVER use `fetch dt.entity.*` for
    SERVICE / HOST / CLOUD_APPLICATION. NEVER use `filter dt.entity.service/host ==`
    on spans, logs, or events. Use `dt.smartscape.*`, `smartscapeNodes`,
    `getNodeName()`, `getNodeField()`. Exception: `fetch dt.entity.process_group`
    ONLY for softwareTechnologies / cmdline metadata (no Smartscape equivalent).

18. **Sub-skills are DQL authority.** dt-deploy-risk carries no comparison
    DQL beyond Phase 0b bootstrap and Phase 1.5 confirmations. Workers MUST
    read their ONE targeted reference file ONCE at start and derive queries
    from it. Hardcoding RED / exception / log / dependency DQL inside this
    skill is prohibited.

19. **Process-metric scope MUST use `dt.entity.process_group_instance`.**
    Filtering runtime metrics on `dt.process_group.id` is forbidden — that
    dimension is frequently empty and produces false "not instrumented"
    findings.

20. **Deploy event resolution is required.** No scorecard without a confirmed
    deployment event. The Phase 0b.0 Substrate Probe selects the live event
    kind/type from `[SDLC_EVENT/task.deployment.finished (primary),
    CUSTOM_DEPLOYMENT, DEPLOYMENT_EVENT]`. For the SDLC_EVENT candidate,
    multiple events for the same `component.name`/`component.version` are
    NORMAL (deploys recur) — take the most recent as T0. For legacy
    candidates, >1 distinct `event.id` for the same version within 48h still
    aborts. Never score against a guessed T0. SDLC_EVENT carries no service
    link — the component→service bridge MUST resolve a real `ROOT_SERVICE_ID`
    before the workers dispatch.

21. **Symmetric windows.** BASELINE_WINDOW and POST_WINDOW MUST be the same
    duration (POST may be truncated at `now()`). Comparing 24h baseline to
    1h post is invalid; the Phase 0b coverage check enforces this.

22. **Delta-only worker outputs.** Workers MUST return delta-shaped
    PhaseResults — NEVER raw BASELINE rows + raw POST rows. The orchestrator
    is not the right context for raw query output.

23. **Set-difference is the unit of comparison for categorical signals.**
    Exception types and log patterns are scored on the POST-minus-BASELINE
    set, NOT on absolute POST counts. A 500% increase in a pre-existing
    exception is a signal but does NOT count toward `new_exception_penalty`.

24. **Parallel worker dispatch in single message.** All 4 workers MUST be
    dispatched in ONE orchestrator message with multiple Agent calls.
    Sequential dispatch defeats the architecture. Workers MUST NOT read
    multiple reference files, re-read files, or read while querying — read
    once, then query in parallel.

25. **Two queries per worker per metric — parallel.** Each worker fires the
    BASELINE query and the POST query in the SAME message. Sequential
    same-worker windows roughly doubles wall time.

### New in dt-deploy-risk — scoring integrity

26. **Penalty cap.** No penalty exceeds 100. No score falls below 0 or
    exceeds 100. The rubric is the ONLY scoring path — no ad-hoc tweaks,
    no "this feels worse than 75".

27. **Partial-window downgrade, never auto-rollback.** If POST_COVERAGE_PCT
    < 50, downgrade the verdict by one tier (GO -> HOLD) but NEVER promote
    HOLD to ROLLBACK from partial data. Auto-recommending a rollback on a
    sparse window is worse than the original bad deploy.

28. **Gap penalty = 50, flagged.** If a worker returns `<gap>` after one
    re-dispatch, set its penalty to 50 and flag in the report header. Never
    silently treat a gap as a 0-penalty (clean signal) or a 100-penalty
    (rollback bias).

### New in dt-deploy-risk — data integrity

29. **A worker's empty/error result is never a conclusion.** Workers must
    attach proof for any absence claim. The orchestrator runs Phase 1.5
    confirmation queries before any "no change / no new exceptions / no
    logs" enters the score or the report.

30. **dtctl auth: on-disk tokens, orchestrator-only.** Tokens are stored on
    disk (`DTCTL_TOKEN_STORAGE=file`, never Keychain). ONLY the orchestrator
    ever logs in, and only reactively on `token "..." not found`:
    `dtctl auth login --plain --safety-level readonly`. Workers NEVER log
    in — on an auth error they return `<gap: auth>`. An auth error is a
    retry, NEVER a "no data" finding. Login is interactive-and-orchestrator-only;
    workers never log in or refresh, and the orchestrator never races
    concurrent/non-interactive logins (see Auth Hardening 2026-06-03).

### New in dt-deploy-risk — report structure & output

31. **Emit the canonical skeleton in order.** Use the Phase 4 section order
    verbatim; never reorder or drop sections. Executive Verdict MUST contain
    the verdict callout and the Recommended Action table.

32. **De-duplication — each fact has ONE canonical home.** Full RED metrics
    -> Before / After table only. Exception list -> New Exceptions only.
    Log patterns -> New Log Patterns only. Score weights -> Score Breakdown
    only. Dependency deltas -> Dependency Impact only. Never restate across
    sections.

33. **Header / Footer = dt-rca clones.** H1 + `## Deployment …` H2 +
    `Generated / Analyst / Environment` + compact deploy/window line.
    Footer: `**Links:**` + `*End of Report*`. No tool-version banner.

34. **DQL only in Appendix D.** Never in Recommended Action, Score
    Breakdown, or anywhere in the body.

35. **Filename = `DEPLOY_RISK_{ANCHOR_SLUG}_{DATE}[_SANITIZED]`.** Never a
    dt-rca or dt-rcf-style filename.

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"PDF DIAGRAM RULES". Summary: no Mermaid in PDFs — ASCII art only, inside
plain code blocks, 80-char max width.

---

## MERMAID SYNTAX RULES

Inherits dt-rca's "MERMAID SYNTAX RULES" (`A["Text"]` not `A[\"Text\"]`;
edge labels `-->|"Text"|`; no escaped quotes). PLUS the dt-rcf
label-sanitization rules:

- **No colons in xychart / gantt task names.** `:` is reserved. Rewrite
  `09:06:08Z` -> `09:06`, `charge.js:77` -> `charge.js L77`.
- **Sequence message labels: no literal double-quotes, no second colon.**
  Paraphrase verbatim error text (≤6 words) — full strings go in TABLES,
  not diagram labels.
- **Node labels use `<br/>` for line breaks — NEVER literal `\n`.**

---

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"DIAGRAM SIZING RULES". Summary: graph LR for cascades >4 nodes; 3-4 words
per node; no deeply nested subgraphs.

---

## ERROR HANDLING

### Deploy Event Not Found
```markdown
## Error: Deploy Event Not Found

No deployment event matched the anchor `{ANCHOR_TYPE}: {ANCHOR_VALUE}`. The
Phase 0b.0 Substrate Probe found no rows for ANY candidate
(`SDLC_EVENT`/`task.deployment.finished`, `CUSTOM_DEPLOYMENT`, `DEPLOYMENT_EVENT`).

**Possible causes:**
- Deploy event ingestion not configured (SDLC events via the events.sdlc
  OpenPipeline, the Deployment Events API, or a OneAgent monitoring rule)
- `component.name` / `component.version` (SDLC_EVENT) or the version tag
  (legacy) spelled differently in the event payload
- Deploy occurred outside the 30d event retention window

**Try:** Inspect raw deploy events on this tenant (probe both substrates):
  `fetch events, from:now()-30d | filter event.kind=="SDLC_EVENT" and event.type=="task.deployment.finished" | fields timestamp, event.id, component.name, component.version, deployment.stage | sort timestamp desc | limit 20`
  — and, if empty, the legacy path:
  `fetch events, from:now()-30d | filter event.kind=="CUSTOM_DEPLOYMENT" or event.type=="CUSTOM_DEPLOYMENT" | fields timestamp, event.id, event.name, properties | sort timestamp desc | limit 20`
```

### Service Entity Not Resolved
```markdown
## Error: Service Entity Not Resolved

The deploy event resolved (component `{SERVICE_NAME}` v{VERSION}) but the
component→service bridge could not map it to a Smartscape SERVICE entity, so
the delta workers have no `ROOT_SERVICE_ID` to scope to.

**Possible causes:**
- `component.name` does not appear (even case-/space-insensitively) in any
  SERVICE display name — the deployable component name differs from the
  monitored service name(s).
- Dynatrace MCP is not registered, so the `find_entity_by_name` fallback is
  unavailable.

**Try:** List candidate services and re-anchor on a specific one:
  `smartscapeNodes SERVICE | fieldsAdd lname = lower(name) | filter contains(lname, lower("{SERVICE_NAME}")) or contains(lname, replaceString(lower("{SERVICE_NAME}"), " ", "")) | fields id, name | limit 25`
  then re-run with the precise service display name as the SERVICE: anchor.
```

### Insufficient Post-Deploy Window
```markdown
## Error: Insufficient Post-Deploy Window

T0 = {T0}. Time since T0 = {minutes_elapsed}m. Required minimum for
`--baseline {BASELINE_DURATION}` is 25% = {required_minutes}m.

**Options:**
- Wait until {wait_until_ts} and re-run.
- Re-run with a shorter baseline: `/dt-deploy-risk {ANCHOR} --baseline {smaller_window}`
- Accept the noise risk and re-run with `--baseline 1h` for an early signal.
```

### Worker fails entirely
Orchestrator marks the corresponding component with `gap`, sets its penalty
to 50, notes the gap in the score breakdown, and continues. Does not
auto-retry beyond the single Phase 1 re-dispatch.

### Davis CoPilot unavailable
Generate the report from the rubric alone. Add banner under Executive Verdict:
```
> **Note:** Davis CoPilot synthesis unavailable. The verdict and recommended
> action are derived from the rubric alone. Re-run for narrative analysis
> when Davis is reachable.
```

### Score on the GO/HOLD boundary
A score of exactly 80 resolves to GO. A score of exactly 50 resolves to
HOLD. Boundary scores get an explicit note in the verdict line:
"Verdict: HOLD (score 50 — boundary case; lean toward manual review)".

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously
until the report is saved.

**Do not stop for confirmation between phases. Do not ask questions.
Generate the complete scorecard.**
