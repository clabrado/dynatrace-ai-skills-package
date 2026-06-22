---
name: dt-vuln-blast
description: >-
  Vulnerability Blast Radius — agentic, multi-worker analysis that turns a CVE or
  library coordinate into a prioritized list of impacted production services with
  reachability evidence, traffic + exposure weighting, ownership routing, and an
  optional draft remediation PR. Anchor types: CVE-YYYY-NNNN or LIB:"name@version".
  Composes Dynatrace AppSec records (security.events) with dt-obs-services,
  dt-obs-tracing reachability via call paths, and dt-obs-kubernetes workload +
  ownership labels. Dispatches four parallel workers (W-affected, W-reachability,
  W-ownership, W-exposure), runs an absence-gate, scores each affected service
  on a 0.35/0.25/0.20/0.20 weighted formula, and produces a Markdown report
  (PDF optional via `--pdf`).
  Supports -clean sanitization, --pr to draft a unified-diff dependency upgrade
  patch into a target GitHub repo, and --apply to actually push the drafted PR
  (--apply requires --pr; never implicit). Use when triaging an AppSec finding,
  responding to a published CVE, or planning a coordinated dependency upgrade
  across multiple owning teams. Trigger: "blast radius for CVE-…",
  "who's exposed to log4j 2.14.1", "prioritize fix list for Spring4Shell",
  "draft a remediation PR for CVE-…", "which services reach the vulnerable
  code path", "ownership routing for vuln X".
---

# dt-vuln-blast — Vulnerability Blast Radius

Agentic vulnerability impact-analysis skill. Accepts a CVE or library coordinate,
dispatches parallel workers backed by maintained DQL sub-skills, scores each
affected service by reachability, traffic, internet-exposure, and error-path
involvement, and produces a prioritized fix list (Markdown; PDF optional via
`--pdf`). Optional
--pr / --apply mode drafts and (only with both flags) pushes a dependency
upgrade PR via the GitHub MCP.

---

## Usage

```
/dt-vuln-blast CVE-2021-44228                                    # CVE anchor (Log4Shell)
/dt-vuln-blast LIB:"org.apache.logging.log4j:log4j-core@2.14.1"  # Library coord anchor
/dt-vuln-blast CVE-2022-22965 -clean                             # Sanitized report (Spring4Shell)
/dt-vuln-blast CVE-2021-44228 --pr REPO:"acme/payment-api"       # Draft remediation PR (no push)
/dt-vuln-blast CVE-2021-44228 --pr REPO:"acme/payment-api" --apply  # Draft AND push the PR
/dt-vuln-blast LIB:"lodash@4.17.20" -clean --pr REPO:"acme/web"  # Sanitized report + drafted PR
/dt-vuln-blast CVE-2021-44228 --pdf                             # Markdown + also render a PDF
```

> **Output:** Markdown is the canonical deliverable and is always written. PDF is
> opt-in via `--pdf` (default OFF) and is best-effort — a missing render engine
> logs one line and the run continues; it never blocks or fails the run.

## Arguments

- `$ARGUMENTS` — exactly one vulnerability anchor (see types below)
- `-clean` — sanitize all identifying names (same rules as `/dt-rca` Phase 1.14)
- `--pr REPO:"owner/repo"` — locate the named GitHub repo, identify the dependency
  manifest, and DRAFT a unified-diff upgrade patch saved next to the report at
  `{report_dir}/{cve}-upgrade.patch`. NOTHING is pushed.
- `--apply` — push the drafted PR via `mcp__github__create_pull_request`. REQUIRES
  `--pr` already present in the same invocation. If `--apply` is set without `--pr`,
  the skill prints an error and STOPS — never implicit writes.

> **Speed note:** run at **medium effort** (not high). High effort multiplies
> latency across all subagents and the report writer; quality is set by reference
> reads + AppSec record completeness, not effort level. Set effort in the harness
> per-run.

---

## Anchor Types

| Anchor | Recognizer | Example |
|--------|-----------|---------|
| CVE | `CVE-\d{4}-\d+` (any position) | `CVE-2021-44228` |
| LIBRARY | `LIB:"<coord>"` where `<coord>` is `group:name@version` (maven), `name@version` (npm/PyPI), or full purl | `LIB:"org.apache.logging.log4j:log4j-core@2.14.1"`, `LIB:"lodash@4.17.20"` |

Library coordinates MUST be normalized to **purl** (`pkg:maven/<group>/<name>@<version>`,
`pkg:npm/<name>@<version>`, `pkg:pypi/<name>@<version>`, `pkg:golang/<module>@<version>`)
or CPE before the AppSec lookup runs. The normalization happens in Phase 0b.

---

## Sub-Skills Loaded Per Phase

dt-vuln-blast carries NO production DQL beyond bootstrap. Each worker reads ONE
targeted reference file (its authority) at start, once per subagent, then derives
queries from it using the dt-vuln-blast scoping rules. This keeps DQL current
without drift — if a field or metric changes upstream, only the reference needs
fixing.

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-affected     | `~/.agents/skills/dt-obs-services/SKILL.md` (entity resolution + Smartscape patterns) |
| W-reachability | `~/.agents/skills/dt-obs-tracing/references/failure-detection.md` + `database-spans.md` + `http-spans.md` |
| W-ownership    | `~/.agents/skills/dt-obs-kubernetes/references/labels-annotations.md` + `workload-health.md` |
| W-exposure     | `~/.agents/skills/dt-obs-services/references/service-metrics.md` (RED + traffic) |

---

## Skill Registry

**Maintenance-time reference map** — the authoritative source for where each
inline query's DQL came from. Used when updating/re-validating the skill, and as
a per-query fallback if a field or metric name errors at run time. NOT read
during normal runs. **Never guess a path, metric, or field name — fix it here
and re-validate.**

### DQL Authority

| File | Purpose | When to read |
|------|---------|--------------|
| `~/.agents/skills/dtctl/references/DQL-reference.md` | Core DQL syntax, filter patterns, aggregation | Always (Phase 0c) |
| `~/.agents/skills/dt-dql-essentials/references/dql/dql-functions-smartscape.md` | smartscapeNodes, getNodeName, getNodeField exact signatures | Always (Phase 0c) |

### W-affected — Dynatrace AppSec (Grail security events)

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-services/SKILL.md` | Always — entity resolution patterns |

> **AppSec schema authority (CORRECTED + VALIDATED LIVE 2026-06-03, tenant
> `demo`):** `security.events` is queried directly. Fields used:
> `event.kind` (== `"SECURITY_EVENT"` — the ONLY kind present; the legacy
> `VULNERABILITY_STATE_REPORT_EVENT` yields 0 rows),
> `vulnerability.references.cve` (ARRAY of CVE strings — match with
> `in("{CVE_ID}", vulnerability.references.cve)`; the legacy scalar
> `vulnerability.cve.id` is 0/absent),
> `vulnerability.cvss.base_score` (float; legacy `vulnerability.cvss.score`
> absent),
> `vulnerability.davis_assessment.level` (severity string e.g. "HIGH"; legacy
> `vulnerability.severity` absent),
> `vulnerability.title`,
> `vulnerability.affected_library.purl` / `.name` / `.version`,
> `affected_entity.id` (the affected entity field — values are
> **`PROCESS_GROUP-*`**, NOT `SERVICE-*`; legacy `affected_entity_ids` and
> `smartscape.affected_entity.ids` are both 0/absent).
>
> **AppSec data source is `security.events` (no `dt.` prefix).** All field/kind
> values above are discovered at runtime by the Phase 0b.0 Substrate Probe and
> interpolated — the literals here are the validated PRIMARY candidates, not
> hardcoded assumptions.
>
> **Aggregate hygiene:** any summary or top-N grouped by an identifier MUST
> guard nulls (e.g. `filter isNotNull(affected_entity.id)`) — `security.events`
> include records without the field populated (provider-meta / subset-change
> events) that pollute aggregates and produce phantom "null" rows. The
> proven CVE query's `collectDistinct(affected_entity.id)` returns a trailing
> `null` — drop it when building `AFFECTED_ENTITY_IDS`.

### W-reachability — dt-obs-tracing

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-tracing/SKILL.md` | Always — W-reachability |
| `~/.agents/skills/dt-obs-tracing/references/failure-detection.md` | Always — error-path involvement |
| `~/.agents/skills/dt-obs-tracing/references/database-spans.md` | If LIB suggests DB driver / ORM (jackson-databind, hibernate, jdbc) |
| `~/.agents/skills/dt-obs-tracing/references/http-spans.md` | If LIB suggests HTTP layer (jetty, tomcat-embed-core, netty, axios) |

### W-ownership — dt-obs-kubernetes

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-kubernetes/SKILL.md` | Always — W-ownership |
| `~/.agents/skills/dt-obs-kubernetes/references/labels-annotations.md` | Always — team/owner label conventions |
| `~/.agents/skills/dt-obs-kubernetes/references/workload-health.md` | Always — workload → service correlation |

### W-exposure — dt-obs-services

| File | When to read |
|------|--------------|
| `~/.agents/skills/dt-obs-services/references/service-metrics.md` | Always — RED metrics + traffic timeseries |

---

**Sanitization, URL format, Mermaid / ASCII diagram rules, and PDF generation:**
These are identical to `/dt-rca`. Read `~/.claude/skills/dt-rca/SKILL.md`
sections "Phase 1.14", "Phase 3", "Phase 4", "PDF DIAGRAM RULES",
"MERMAID SYNTAX RULES", and "DIAGRAM SIZING RULES" — all apply unchanged.

---

## ENTITY MODEL — SMARTSCAPE-NATIVE (NON-NEGOTIABLE)

`classicEntitySelector()` is explicitly deprecated. dt-vuln-blast uses zero
`dt.entity.*` references for span / log / event filtering. The Smartscape model
applies throughout — identical to `/dt-rcf`.

### Span / Log / Event Filtering

The AppSec affected entities are **`PROCESS_GROUP-*`** (validated live
2026-06-03). Spans/logs scope by process-group identity, NEVER by
`dt.entity.*  ==`.

<!-- VALIDATED LIVE 2026-06-03: data source `security.events` confirmed;
     affected_entity.id holds PROCESS_GROUP-* values; CVE match via
     in("{CVE_ID}", vulnerability.references.cve). Span scoping uses the
     standard dimension dt.entity.process_group — `smartscapeNodes
     PROCESS_GROUP` returns 0 nodes and `dt.smartscape.process_group` is NULL
     on spans for this tenant. -->
```dql
-- Filter spans to a specific affected PROCESS_GROUP (standard dimension)
fetch spans
| filter dt.entity.process_group == "{AFFECTED_PG_ID}"

-- PG→SERVICE bridge: the serving service(s) for friendly labels
fetch spans
| filter dt.entity.process_group == "{AFFECTED_PG_ID}"
| summarize services = collectDistinct(dt.smartscape.service)

-- Filter security events to a specific affected PROCESS_GROUP (used by W-affected)
fetch security.events
| filter event.kind == "{EVENT_KIND}"          -- probe-bound, e.g. SECURITY_EVENT
| filter affected_entity.id == "{AFFECTED_PG_ID}"
```

### Process Metrics Scoping

Process and runtime metrics ALWAYS scope by `dt.entity.process_group_instance`
(a standard metric dimension), NEVER `dt.process_group.id` (frequently EMPTY and
yields false 0-row results — the iteration-F failure documented in dt-rcf).

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
timeseries jvm_loaded = avg(dt.runtime.jvm.classloader.loaded_class_count),
           by: { dt.entity.process_group_instance }
| filter dt.entity.process_group_instance == "{AFFECTED_PGI}"
```

### Entity Name & Metadata Resolution

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
-- Resolve service name on spans (preferred over fetch dt.entity.service)
fetch spans
| fieldsAdd service_name = getNodeName(dt.smartscape.service)

-- Look up service metadata directly
smartscapeNodes SERVICE
| filter id == "{AFFECTED_SERVICE_ID}"
| fields name, properties
```

### Time Field Discipline

- **Logs filter on `timestamp`**, NEVER `start_time`.
- **Spans filter on `start_time`**, NEVER `timestamp`.
- **Security events filter on `timestamp`** (same as logs).
- ALIAS every `bin()`:
  `summarize n=count(), by:{ ts = bin(timestamp, 1h) } | sort ts asc`
  NEVER `sort \`bin(...)\`` — backticks break in `dtctl query '...'` and re-referencing
  `bin(timestamp)` after summarize errors `field timestamp doesn't exist`.

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking — set up at the very start)

Maintain a per-run timing log so every run leaves diagnosable phase timing
behind. Set `LOG="VULN_{ANCHOR_SLUG}_{DATE}.log"` (next to the report) and at
each phase boundary append ONE line, **backgrounded so it never gates execution**:
```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line summary>" >> "$LOG"; } &
```
Log: START, Phase 0b complete (AppSec record + affected count), Phase 1 complete
(per-worker one-liners), Absence-Gate result, Phase 2 prioritization complete
(top-3 services + scores), Phase 3 Davis synthesis, report written, PR drafted
(if `--pr`), PR pushed (if `--apply`), RUN COMPLETE. Phase 6 prints the log path.

---

### Phase 0a: Anchor Parser

```
ANCHOR_TYPE + ANCHOR_VALUE:
  CVE-\d{4}-\d+ pattern (any position)   → CVE
  LIB:"<coord>" or LIB:<coord>           → LIBRARY

FLAGS:
  -clean                  → CLEAN_MODE = true
  --pr REPO:"<owner/repo>" → PR_MODE = true, PR_REPO = "<owner/repo>"
  --apply                 → APPLY_MODE = true
  --pdf                   → PDF_MODE = true   (default FALSE — Markdown only)
  pdf=true                → alias for --pdf (PDF_MODE = true)
  pdf=false               → no-op (PDF stays off; legacy alias, accepted silently)

VALIDATION:
  - if no anchor parseable → print usage and exit
  - if APPLY_MODE == true and PR_MODE == false → print error and exit:
    "ERROR: --apply requires --pr. Re-run with --pr REPO:\"owner/repo\" --apply."
  - if more than one anchor → print error and exit (single anchor only)

DEFAULTS:
  - PDF_MODE = false. Markdown is the canonical deliverable and is ALWAYS
    written. PDF is opt-in via --pdf and is best-effort / non-fatal.
```

---

### Phase 0-auth: PRE-WARM the dtctl session (Orchestrator — proactive step 0)

**FIRST action of the whole run — before any data query and before any dispatch:**
```
dtctl auth refresh
```
This is a REFRESH, not a login — silent, no browser, uses the on-disk refresh
token (`DTCTL_TOKEN_STORAGE=file`, harness env, never the Keychain). It
guarantees a fresh access token up front, which (a) avoids the failed-query →
status → refresh → re-run thrash, and (b) means the parallel workers all
inherit one fresh token instead of each triggering a concurrent refresh (a race).
Pre-warming is consistent with "never proactively LOG IN" — refresh ≠ login.

- `dtctl auth refresh` succeeds → token fresh for the entire run; proceed to Phase 0b.
- It fails (no/expired refresh token) → ONLY THEN run once:
  `dtctl auth login --plain --safety-level readonly`.

**Workers never authenticate** — they inherit the pre-warmed on-disk token. If a
worker still returns `<gap: auth>`, the orchestrator refreshes once and
re-dispatches only that worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

---

### Phase 0b: AppSec Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** that all workers receive. **MINIMAL BOOTSTRAP —
1 probe + 2 lookup queries.** Bootstrap is the orchestrator's only serial
stretch; keep it tiny.

#### Step 0 — Substrate Probe (runs ONCE, before everything else)

**Root cause of past failures:** the bootstrap hardcoded tenant-specific
`event.kind` values and `vulnerability.*` / affected-entity field names from a
2026-05-28 snapshot that DRIFTED. The reference-driven workers survive drift;
this orchestrator bootstrap did not. Fix: discover the real shape at runtime,
then interpolate the discovered values into the lookup queries — never assume a
stale literal. Make the **corrected 2026-06-03 value the PRIMARY candidate** so
the skill works today and self-heals tomorrow.

Run these cheap probes (combine where possible) and bind the results into
variables the Step-2 lookup interpolates:

**(a) Which `event.kind` value carries vulnerability records?**
```dql
fetch security.events, from:now()-3d
| summarize c = count(), by: { event.kind }
| sort c desc
| limit 20
```
Candidate list (ordered `[corrected-2026-06-03, legacy]`):
`EVENT_KIND ∈ ["SECURITY_EVENT", "VULNERABILITY_STATE_REPORT_EVENT"]`.
Bind `EVENT_KIND` = the first candidate present in the probe output.

**(b) Which CVE field is populated?**
```dql
fetch security.events, from:now()-3d
| filter event.kind == "{EVENT_KIND}"
| summarize ref = countIf(isNotNull(vulnerability.references.cve)),
            legacy = countIf(isNotNull(vulnerability.cve.id))
```
Candidate list: `CVE_FIELD ∈ ["vulnerability.references.cve" (array → use
in(CVE, field)), "vulnerability.cve.id" (scalar → use field == CVE)]`.
Bind `CVE_FIELD` + its match form to the first non-zero column.

**(c) Which affected-entity field is populated, and what entity type?**
```dql
fetch security.events, from:now()-3d
| filter event.kind == "{EVENT_KIND}"
| summarize aff = countIf(isNotNull(affected_entity.id)),
            legacy = countIf(isNotNull(affected_entity_ids)),
            ss = countIf(isNotNull(smartscape.affected_entity.ids))
```
Candidate list: `ENTITY_FIELD ∈ ["affected_entity.id", "affected_entity_ids",
"smartscape.affected_entity.ids"]`. Bind `ENTITY_FIELD` = first non-zero column.
The values are **`PROCESS_GROUP-*`** on the current tenant (NOT `SERVICE-*`) —
see VB-3 / the PROCESS_GROUP entity model below; workers scope off PROCESS_GROUP.

**Emit a one-line probe summary to the run log**, e.g.:
`Probe: event.kind→SECURITY_EVENT, cve field→vulnerability.references.cve (array),
entity field→affected_entity.id (PROCESS_GROUP)`.

**If NONE of the candidates resolve for a given dimension** (every column zero /
no kind present), THEN — and only then — emit the "No Affected Entities" /
"AppSec not enabled" banner (Error Handling). Absence is gated by the probe,
never assumed by a stale literal.

> **VALIDATED LIVE 2026-06-03 (tenant `demo`, anchor `CVE-2023-44487`):**
> probe (a) → only `event.kind` is `SECURITY_EVENT` (21.5M over 3d); probe (b) →
> `vulnerability.references.cve` populated (9.0M), `vulnerability.cve.id` = 0;
> probe (c) → `affected_entity.id` populated (1.75M), both legacy forms = 0.

#### Step 1 — Normalize LIBRARY coordinate to purl (skip if ANCHOR_TYPE == CVE)

Convert `LIB:"<coord>"` to a purl string. Detection by character pattern:

| Input form | Heuristic | Normalized purl |
|------------|-----------|------------------|
| `group:name@version` (contains `:`) | maven (Java) | `pkg:maven/<group>/<name>@<version>` |
| `name@version` starting with `@scope/` | npm scoped | `pkg:npm/@scope/<name>@<version>` |
| `name@version` (no `:` and no `/`) | npm or PyPI — try both | `pkg:npm/<name>@<version>` (primary), `pkg:pypi/<name>@<version>` (fallback) |
| `module@version` containing `/` and no `:` | Go module | `pkg:golang/<module>@<version>` |
| Already starts with `pkg:` | already a purl | pass through |

Store as `LIB_PURL`. If ambiguous (npm vs PyPI), the affected query in Step 2
matches on EITHER.

#### Step 2 — AppSec vulnerability record lookup

**Interpolate the Step-0 probe variables** (`{EVENT_KIND}`, `{CVE_FIELD}`,
`{ENTITY_FIELD}`) into these queries — do NOT hardcode the literals. The
queries below show the **corrected 2026-06-03 PRIMARY values** as the working
example; the probe binds whichever candidate is live.

<!-- VALIDATED LIVE 2026-06-03 (tenant demo, CVE-2023-44487 → CVSS 7.5 HIGH,
     19 affected PROCESS_GROUPs, 326K events). event.kind==SECURITY_EVENT;
     CVE via in(CVE, vulnerability.references.cve) [array]; entities in
     affected_entity.id (PROCESS_GROUP-*); severity via
     vulnerability.davis_assessment.level; score via vulnerability.cvss.base_score. -->
```dql
-- ANCHOR_TYPE == CVE   ({EVENT_KIND}=SECURITY_EVENT, {ENTITY_FIELD}=affected_entity.id)
fetch security.events, from:now()-30d
| filter event.kind == "{EVENT_KIND}"
| filter in("{CVE_ID}", vulnerability.references.cve)
| summarize severity            = takeFirst(vulnerability.davis_assessment.level),
            cvss_score          = takeFirst(vulnerability.cvss.base_score),
            title               = takeFirst(vulnerability.title),
            affected_libraries  = collectDistinct(vulnerability.affected_library.purl),
            affected_entity_ids = collectDistinct(affected_entity.id)
| fields severity, cvss_score, title, affected_libraries, affected_entity_ids
```
> If the probe binds the legacy scalar CVE field, swap line 3 to
> `| filter vulnerability.cve.id == "{CVE_ID}"`. If it binds a legacy entity
> field, swap `collectDistinct(affected_entity.id)` to that field.

<!-- VALIDATED LIVE 2026-06-03: same substrate as CVE path; library match uses
     vulnerability.affected_library.purl with name/version fallback. -->
```dql
-- ANCHOR_TYPE == LIBRARY (purl match, with name/version fallback)
fetch security.events, from:now()-30d
| filter event.kind == "{EVENT_KIND}"
| filter vulnerability.affected_library.purl == "{LIB_PURL}"
       or (vulnerability.affected_library.name == "{LIB_NAME}"
           and vulnerability.affected_library.version == "{LIB_VERSION}")
| summarize cves                = collectDistinct(vulnerability.references.cve),
            severity            = takeFirst(vulnerability.davis_assessment.level),
            cvss_score          = takeMax(vulnerability.cvss.base_score),
            title               = takeFirst(vulnerability.title),
            affected_libraries  = collectDistinct(vulnerability.affected_library.purl),
            affected_entity_ids = collectDistinct(affected_entity.id)
| fields cves, severity, cvss_score, title, affected_libraries, affected_entity_ids
```
> `vulnerability.references.cve` is an ARRAY — `collectDistinct` over it yields
> a nested collection; flatten in the orchestrator when building `CVE_IDS[]`.

#### Extract DispatchContext

```
CVE_IDS[]              CVEs from vulnerability.references.cve (collected/flattened — may be N for a library)
SEVERITY               CRITICAL | HIGH | MEDIUM | LOW  (from vulnerability.davis_assessment.level)
CVSS_SCORE             float  (from vulnerability.cvss.base_score)
TITLE                  vulnerability.title
AFFECTED_LIBRARIES[]   purl strings
AFFECTED_ENTITY_IDS[]  PROCESS_GROUP-XXXX entity IDs (from affected_entity.id — the
                       AppSec affected entities are PROCESS_GROUPs on this tenant,
                       NOT services; drop any null entry the collect may include)
AFFECTED_COUNT         length(AFFECTED_ENTITY_IDS)
DERIVED_TIME_WINDOW    from: now()-7d, to: now()   (traffic + reachability window)
```

If `AFFECTED_COUNT == 0`: emit the "No affected entities" error report
(Error Handling section) and exit. Do NOT proceed to Phase 1.

#### ENTITY MODEL DECISION (VB-3) — workers scope off PROCESS_GROUP

The AppSec affected entities are **`PROCESS_GROUP-*`**, not `SERVICE-*`
(validated live 2026-06-03). **Decision for this skill: re-scope the four
workers (W-affected / W-reachability / W-ownership / W-exposure) off
PROCESS_GROUP as the PRIMARY entity model**, because that is the entity type the
AppSec substrate actually emits and it requires no unvalidated topology
traversal. **Span/metric scoping uses standard dimensions — `smartscapeNodes
PROCESS_GROUP` returns 0 nodes and `dt.smartscape.process_group` is NULL on
spans for this tenant (validated 2026-06-03), so neither is usable:**
- Spans/logs: `filter dt.entity.process_group == "{pg_id}"` (VALIDATED LIVE —
  254K spans / 116K root spans for a real affected PG).
- Process/runtime metrics: scope by `dt.entity.process_group_instance` (a
  standard metric dimension that hangs off the PG), NEVER `dt.process_group.id`.

**Optional SERVICE enrichment (best-effort, never gating) — VALIDATED bridge:**
a span query scoped by `dt.entity.process_group == "{pg_id}"` exposes the
serving SERVICE via `collectDistinct(dt.smartscape.service)` (e.g.
`PROCESS_GROUP-8ADC…` → `SERVICE-22E7…` = `hipstershop.AdService`). The
orchestrator MAY enrich each PROCESS_GROUP with its serving SERVICE name for
friendlier report labels. This enrichment is cosmetic — if it returns nothing,
the report uses the PROCESS_GROUP id. NEVER block a run or drop an affected
entity because the bridge was empty.

#### Resolve PROCESS_GROUP names + cluster context (span-derived, batched)

`smartscapeNodes PROCESS_GROUP` is empty on this tenant, so resolve names and
cluster context from spans scoped by the affected process groups, and (optional)
bridge to the serving SERVICE name.

<!-- VALIDATED LIVE 2026-06-03: dt.entity.process_group scoping returns spans;
     dt.smartscape.service / k8s.* fields are span-side. -->
```dql
fetch spans, from:now()-7d
| filter in(dt.entity.process_group, {AFFECTED_ENTITY_IDS_ARRAY})
| summarize service_ids   = collectDistinct(dt.smartscape.service),
            k8s_cluster   = takeFirst(k8s.cluster.name),
            k8s_namespace = takeFirst(k8s.namespace.name),
            k8s_workload  = takeFirst(k8s.workload.name),
            by: { pg_id = dt.entity.process_group }
-- then (optional) name each service_id via:
--   smartscapeNodes SERVICE | filter id == "{service_id}" | fields id, name
```

Store as `AFFECTED_SERVICES[] = [ { id, name, k8s_cluster?, k8s_namespace? }, … ]`
(name is the PROCESS_GROUP name, optionally enriched with serving SERVICE name).

Store everything as **DispatchContext JSON**:

```json
{
  "ANCHOR_TYPE": "CVE|LIBRARY",
  "ANCHOR_VALUE": "...",
  "CVE_IDS": ["CVE-YYYY-NNNN", "..."],
  "SEVERITY": "CRITICAL|HIGH|MEDIUM|LOW",
  "CVSS_SCORE": 9.8,
  "AFFECTED_LIBRARIES": ["pkg:maven/..."],
  "LIB_PURL": "pkg:maven/... or null",
  "AFFECTED_ENTITY_IDS": ["SERVICE-XXXX"],
  "AFFECTED_SERVICES": [{ "id": "SERVICE-XXXX", "name": "...", "k8s_cluster": "...", "k8s_namespace": "..." }],
  "AFFECTED_COUNT": 0,
  "DERIVED_TIME_WINDOW": { "from": "ISO", "to": "ISO" },
  "CLEAN_MODE": false,
  "PR_MODE": false,
  "PR_REPO": "owner/repo or null",
  "APPLY_MODE": false,
  "PDF_MODE": false
}
```

---

### Phase 0c: Reference Strategy

dt-vuln-blast is **reference-driven** — workers carry no hardcoded production
DQL. Each worker reads ONE targeted reference file at start (see Sub-Skills
table), once per subagent, derives the query patterns from it, and applies the
dt-vuln-blast scoping below. This is the correctness design: DQL stays current
automatically, drift is impossible.

**Orchestrator exceptions (inline only):** Phase 0b bootstrap queries and Phase
1.5 Absence-Gate confirmation queries remain inline — they are orchestration
primitives, tiny, stable, and must run BEFORE workers dispatch.

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
concurrency.** Dispatching them one-at-a-time (send, wait, send) defeats the
architecture and is a top cause of slow runs. The whole point is that the four
workers run AT THE SAME TIME.

Workers W-affected, W-reachability, W-ownership, W-exposure run concurrently as
subagents (`subagent_type: Explore`).

**Speed model:** each worker reads ONE targeted reference file (5–20KB, once),
derives its queries, and fires them in parallel. By the time you dispatch, the
orchestrator's Phase 0b queries have already pre-warmed the on-disk dtctl token,
so workers never block on auth.

**Worker-failure handling — NO silent serial fallback.** If a worker returns
gaps or fails, do NOT re-run all of its queries yourself inline in the
orchestrator thread — that serializes the work AND pulls raw query output into
the orchestrator context. Instead: **re-dispatch that ONE worker ONCE** (as a
fresh Agent call, with a note on what to fix), and if it still fails, record a
`<gap: ...>` marker for that worker in Phase 2 scoring.

#### Worker Prompt Template

```
You are a vulnerability-impact worker for dt-vuln-blast.

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed in "REFERENCE FILES" below. Read each file
ONCE. Do not read any other files. Do not re-read. Take the query patterns
from the reference; apply the DISPATCH CONTEXT scoping (entity IDs, time window,
dtctl hygiene rules). This is the ONLY source of DQL truth — do NOT derive
queries from training knowledge or guess field names.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple `dtctl query` tool
calls, in parallel). Do not read files and query at the same time — read
first, then query.

ENTITY MODEL — CRITICAL (affected entities are PROCESS_GROUP-*, per VB-3):
- The affected entities are `PROCESS_GROUP-*`. On this tenant `smartscapeNodes
  PROCESS_GROUP` returns 0 nodes and `dt.smartscape.process_group` is NULL on
  spans — so DO NOT use those. Scope spans to a PROCESS_GROUP with the standard
  dimension (VALIDATED LIVE 2026-06-03):
  filter dt.entity.process_group == "PROCESS_GROUP-ID"
- NEVER use dt.entity.service/host == "..." or classicEntitySelector().
- For process/runtime metrics: scope by `dt.entity.process_group_instance`
  (a standard metric dimension). `dt.process_group.id` is frequently EMPTY —
  NEVER filter on it.
- PG→SERVICE bridge (validated): a span query scoped by
  `dt.entity.process_group == "{pg_id}"` exposes the serving service(s) via
  `dt.smartscape.service` — `collectDistinct(dt.smartscape.service)` yields the
  SERVICE id(s); `getNodeName(dt.smartscape.service)` (or `smartscapeNodes
  SERVICE | filter id ==`) yields the name for friendly report labels.
- Logs filter on `timestamp`, spans on `start_time`. Alias every `bin()`.

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

QUERIES TO RUN:
{QUERY_INSTRUCTIONS_FOR_THIS_WORKER}

RULES:
- EXPLORATION CAP: run ONLY the queries defined in "QUERIES TO RUN" below.
  No ad-hoc "let me also check…" queries. If a finding requires one additional
  query, use a pattern from your already-loaded reference — never a free guess.
- PREFLIGHT: run
  `DTCTL_TOKEN_STORAGE=file dtctl query 'fetch spans | limit 1'`.
  If it errors with an auth/connection problem (not a DQL error), return
  immediately with `<gap: dtctl-unavailable: {message}>` and STOP — do NOT
  emit "no data / not present" findings.
- Run all independent queries in parallel (multiple tool calls per message).
- Do NOT return raw rows. Internalize; summarize aggressively.
- Cap your WorkerResult at ~4KB. Per-service rollups, not per-span dumps.
- If a query fails with a SYNTAX/FIELD error, retry ONCE with corrected
  syntax (check: aliased bin not backticked; `timestamp` for logs vs
  `start_time` for spans; single-quoted dtctl arg).
- AUTH IS THE ORCHESTRATOR'S JOB — workers NEVER run `dtctl auth login` OR
  `dtctl auth refresh`. If a dtctl call returns an auth error, verify EVERY
  dtctl call has the `DTCTL_TOKEN_STORAGE=file` prefix. If the prefix is
  present and it still fails, return `<gap: auth>` and STOP — never attempt
  any auth recovery.
- ERROR ≠ ABSENCE: a query that ERRORED tells you nothing about whether data
  exists. Only a SUCCESSFUL zero-row query is evidence of absence — and if
  that query was scoped/filtered, CONFIRM with a broader unfiltered query
  before concluding "not reached" / "not exposed" / "no owner".
- PROOF OF ABSENCE (required): if you DO conclude "not reached / not exposed
  / no owner", your WorkerResult MUST include the exact UNFILTERED
  confirmation query you ran and its zero-row result.
- Execute DQL via
  `DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'`
  — prefix EVERY dtctl call with `DTCTL_TOKEN_STORAGE=file` (no space). Use
  SINGLE QUOTES around the DQL, always — double quotes let bash interpret
  `$`, backticks, and `"` inside the query.
- NO BACKTICKS in DQL. For binned timeseries, ALIAS the bin
  (`by:{ ts = bin(start_time, 15m) } | sort ts asc`) — never
  `sort \`bin(...)\``.

RETURN only this WorkerResult shape (JSON). Nothing else.
{WORKER_RESULT_SCHEMA}
```

---

#### W-affected — enumerate affected components per CVE/lib

**REFERENCE-DRIVEN — dt-vuln-blast carries NO inline AppSec DQL.** Read
`~/.agents/skills/dt-obs-services/SKILL.md` ONCE for entity resolution patterns.
Take the AppSec record patterns from the **Skill Registry > W-affected** schema
notes above, applying these dt-vuln-blast must-keeps.

**dt-vuln-blast scoping/hygiene for every W-affected query (CORRECTED
2026-06-03 — use the Phase 0b.0 probe-bound values, shown here as the validated
PRIMARY literals):**
- Filter `event.kind == "{EVENT_KIND}"` on `security.events`
  (`EVENT_KIND` = `SECURITY_EVENT` on this tenant).
- Match the CVE via `in("{CVE_ID}", vulnerability.references.cve)` (the CVE field
  is an ARRAY) — for N CVEs, OR the per-CVE `in(...)` predicates. Match the
  library via `vulnerability.affected_library.purl in {AFFECTED_LIBRARIES}`.
- Affected entities live in `affected_entity.id` and are **`PROCESS_GROUP-*`**.
  Group/collect by `affected_entity.id`; guard `isNotNull(affected_entity.id)`.
- Severity = `vulnerability.davis_assessment.level`; score =
  `vulnerability.cvss.base_score`; title = `vulnerability.title`.
- 30-day window: `from:now()-30d`. Filter on `timestamp`.
- Single-quote the dtctl arg.

**Queries to run (ALL in one parallel batch). Group by `affected_entity.id`
(PROCESS_GROUP):**

1. **Per-PROCESS_GROUP affected component breakdown** — affected libraries
   observed on each entity in `AFFECTED_ENTITY_IDS`. Group by
   `affected_entity.id`. Report `affected_library_versions[]` per process group
   (multi-version installs are common — a PG may carry `log4j-core 2.14.1` AND
   `2.17.1`).
2. **Vulnerability lifecycle status** — for each entity, take the most recent
   `vulnerability.resolution` / `vulnerability.management_state` field; flag
   process groups already in `IGNORED` or `RESOLVED` state.
3. **First-seen / last-seen timestamps** — when did the AppSec finding first
   appear per process group; this informs deployment correlation in Phase 3.

**WorkerResult shape** (`service_id`/`service_name` carry the PROCESS_GROUP id +
name; the report labels these as the affected entity, optionally enriched with a
serving SERVICE name):
```json
{
  "per_service": [
    {
      "service_id": "PROCESS_GROUP-XXXX",
      "service_name": "...",
      "affected_library_versions": [{ "purl": "...", "version": "..." }],
      "vulnerability_state": "MUTED|OPEN|RESOLVED|IGNORED",
      "first_seen": "ISO",
      "last_seen": "ISO"
    }
  ],
  "summary": { "open_count": 0, "resolved_count": 0, "ignored_count": 0 },
  "gaps": []
}
```

---

#### W-reachability — span call paths through the vulnerable library

**REFERENCE-DRIVEN — read `~/.agents/skills/dt-obs-tracing/references/failure-detection.md`
ONCE** (failure reason taxonomy, exception types). Read
`http-spans.md` ONLY if the library suggests HTTP entry (`jetty`, `tomcat-embed-core`,
`netty`, `axios`, `express`). Read `database-spans.md` ONLY if the library
suggests DB/ORM (`jackson-databind`, `hibernate`, `jdbc`, `mysql-connector`).

**dt-vuln-blast scoping/hygiene for every W-reachability query (VB-3:
PROCESS_GROUP-scoped — the affected entities are `PROCESS_GROUP-*`):**
- For EACH `pg_id` in `AFFECTED_ENTITY_IDS`, scope spans with the standard
  dimension `filter dt.entity.process_group == "{pg_id}"` (VALIDATED LIVE
  2026-06-03 — 254K spans / 116K root spans for a real affected PG).
  Do NOT use `smartscapeNodes PROCESS_GROUP` (0 nodes on this tenant) or
  `dt.smartscape.process_group` (NULL on spans).
- Filter `start_time >= toTimestamp("{from}") AND start_time <= toTimestamp("{to}")`.
- ALIAS every bin (`ts = bin(start_time,1h) | sort ts asc`), NEVER backticks.
- Single-quote the dtctl arg.

**Queries to run (per affected service, batched across services):**

1. **HTTP/RPC entry points** — root spans entering the process group:
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     PROCESS_GROUP-scoped per VB-3 (affected entities are PROCESS_GROUP-*). -->
```dql
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| filter request.is_root_span == true
| summarize requests = count(),
            failed   = countIf(request.is_failed == true),
            by: { endpoint.name, http.route, span.kind }
| sort requests desc
| limit 10
```
2. **Error-path involvement** — fraction of failing spans:
<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| summarize total = count(),
            failed = countIf(request.is_failed == true)
| fieldsAdd error_pct = (failed * 100.0) / total
```
3. **Outbound DB callouts** (if the lib is a DB driver / ORM):
<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| filter span.kind == "client" and isNotNull(db.system)
| summarize calls = count(),
            avg_ms = avg(duration)/1e6,
            by: { db.system, db.name, db.operation }
| sort calls desc
| limit 10
```
4. **Reachability evidence** — does production traffic actually exercise the
   library's package? For Java/Node, look for span names whose `code.namespace`
   or `code.function` matches the affected library's package prefix
   (e.g. `org.apache.logging.log4j.*` for Log4Shell). If span schema lacks
   `code.namespace`, fall back to library presence on entity (`per_service`
   from W-affected) as a weaker reachability proxy.

**Reachability score per process group (binary + bonus):**
- `reached = true` if (a) the process group is in `AFFECTED_ENTITY_IDS` AND (b)
  root-span traffic > 0 in the window AND (c) either: span-level code.namespace
  match OR — when code.namespace is absent — the library appears on the entity
  per W-affected.
- `reached = false` if no root-span traffic in the 7d window.
- `traffic_path_pct` (bonus): fraction of total root spans matching the
  library's likely entry pattern (HTTP, DB, RPC). 0–100 float.

**WorkerResult shape** (`service_id` = the affected `PROCESS_GROUP-*` id):
```json
{
  "per_service": [
    {
      "service_id": "PROCESS_GROUP-XXXX",
      "service_name": "...",
      "reached": true,
      "traffic_path_pct": 100.0,
      "entry_points": [{ "endpoint": "...", "requests": 0, "failed": 0 }],
      "error_pct": 0.0,
      "db_callouts": [{ "db_system": "...", "calls": 0 }],
      "evidence_basis": "span.code.namespace|entity-library-presence"
    }
  ],
  "gaps": []
}
```

---

#### W-ownership — services → owning teams via Smartscape + K8s labels

**REFERENCE-DRIVEN — read `~/.agents/skills/dt-obs-kubernetes/references/labels-annotations.md`
+ `workload-health.md` ONCE.** Take label conventions from the reference; apply
the dt-vuln-blast scoping below.

**dt-vuln-blast scoping/hygiene for every W-ownership query (VB-3:
PROCESS_GROUP-scoped):**
- For each `pg_id` in `AFFECTED_ENTITY_IDS`, resolve the K8s workload via spans
  scoped to the PROCESS_GROUP (`k8s.namespace.name`, `k8s.workload.name`,
  `k8s.cluster.name`) — see the dt-rcf bootstrap Q2 pattern. Scope with the
  standard dimension `filter dt.entity.process_group == "{pg_id}"` (NOT
  `smartscapeNodes PROCESS_GROUP`, which is 0 on this tenant).
- For labels and annotations, read from the K8s workload entity via Smartscape.
- Standard owner labels to try, in order of preference:
  `dynatrace.tag.owner`, `k8s.workload.label[\"app.kubernetes.io/owner\"]`,
  `k8s.workload.label[\"owner\"]`, `k8s.workload.annotation[\"team\"]`,
  `k8s.workload.label[\"team\"]`, `k8s.workload.label[\"cost-center\"]`.

**Queries to run (per affected service, batched across services):**

1. **Resolve K8s workload per process group:**
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     PROCESS_GROUP-scoped per VB-3. -->
```dql
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| summarize n = count(),
            by: { k8s.cluster.name, k8s.namespace.name, k8s.workload.name }
| sort n desc
| limit 1
```
2. **Lookup workload labels + annotations:**
<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
smartscapeNodes KUBERNETES_WORKLOAD
| filter getNodeField(@this, "k8s.cluster.name") == "{cluster}"
       and getNodeField(@this, "k8s.namespace.name") == "{namespace}"
       and name == "{workload}"
| fields id, name, properties
```
3. **Fallback — serving SERVICE + its tags (PG→SERVICE bridge, validated):**
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     smartscapeNodes PROCESS_GROUP returns 0 on this tenant — bridge to the
     serving SERVICE via spans, then read the SERVICE node's properties/tags. -->
```dql
-- a) bridge: PROCESS_GROUP → serving SERVICE id(s)
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| summarize services = collectDistinct(dt.smartscape.service)
-- b) read tags/owner labels off the serving SERVICE node
smartscapeNodes SERVICE
| filter id == "{service_id_from_bridge}"
| fields id, name, properties
```

**Owner resolution algorithm (per service):**
- Try labels/annotations in the preference order above; first non-empty wins.
- If all empty → `owner = "UNKNOWN"`, `owner_basis = "none-found"`.
- Record `owner_basis` so the report can flag UNKNOWN owners as a
  routing gap to fix.

**WorkerResult shape** (`service_id` = the affected `PROCESS_GROUP-*` id):
```json
{
  "per_service": [
    {
      "service_id": "PROCESS_GROUP-XXXX",
      "service_name": "...",
      "k8s_cluster": "... or null",
      "k8s_namespace": "... or null",
      "k8s_workload": "... or null",
      "owner": "team-slug or UNKNOWN",
      "owner_basis": "k8s.workload.label[owner]|dynatrace.tag.owner|...|none-found",
      "cost_center": "... or null"
    }
  ],
  "gaps": []
}
```

---

#### W-exposure — traffic volume + internet-facing classification

**REFERENCE-DRIVEN — read
`~/.agents/skills/dt-obs-services/references/service-metrics.md` ONCE.** Take
the RED + traffic timeseries patterns from there; apply the dt-vuln-blast
must-keeps below.

**dt-vuln-blast scoping/hygiene for every W-exposure query (VB-3:
PROCESS_GROUP-scoped):**
- Scope to each affected `pg_id` with the standard dimension
  `filter dt.entity.process_group == "{pg_id}"` (NOT `smartscapeNodes
  PROCESS_GROUP`, which is 0 on this tenant).
- Use SPAN-DERIVED traffic for OTel safety (dt-rcf must-keep): root spans, not
  `dt.service.request.*` metrics (the `dt.service.name` ambiguity bites OTel
  services).
- Filter `start_time` on spans; ALIAS bins; single-quote the dtctl arg.

**Queries to run (per affected service, batched across services):**

1. **Traffic volume (req/s, peak):**
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     PROCESS_GROUP-scoped per VB-3. -->
```dql
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| filter request.is_root_span == true
| summarize reqs = count(),
            by: { ts = bin(start_time, 1h) }
| sort ts asc
```
   Aggregate to `requests_per_hour_avg` and `requests_per_hour_peak`.
2. **Internet-facing classification — bridge PG → serving SERVICE, read edges:**
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     smartscapeNodes PROCESS_GROUP is 0 on this tenant — classify exposure on
     the serving SERVICE node reached via the PG→SERVICE span bridge. -->
```dql
-- a) bridge: PROCESS_GROUP → serving SERVICE id(s)
fetch spans, from:now()-7d
| filter dt.entity.process_group == "{pg_id}"
| summarize services = collectDistinct(dt.smartscape.service)
-- b) classify the serving SERVICE node
smartscapeNodes SERVICE
| filter id == "{service_id_from_bridge}"
| fieldsAdd internet_facing = isNotNull(getNodeField(@this, "publicDomainNames"))
                              or contains(properties, "INTERNET")
| fields id, name, internet_facing, properties
```
3. **Ingress class fallback (Kubernetes workload with Ingress):**
<!-- VALIDATE: run via dtctl query against tenant before first production use.
     Bridge PROCESS_GROUP → its serving SERVICE/KUBERNETES_SERVICE only if the
     relationship is available; best-effort, never gating. -->
```dql
smartscapeNodes KUBERNETES_SERVICE
| filter id == "{k8s_service_id}"   -- from the optional PG→service enrichment
| fields name, properties
```
   Flag `internet_facing = true` if `properties` contains
   `ingressClass` / `LoadBalancer` / public IP.

**Exposure scoring per service:**
- `traffic_score` = `min(1.0, log10(1 + requests_per_hour_peak) / 4.0)` — saturates
  near 10k req/h.
- `internet_score` = `1.0` if internet-facing, else `0.2` (internal services
  still carry lateral risk).

**WorkerResult shape** (`service_id` = the affected `PROCESS_GROUP-*` id):
```json
{
  "per_service": [
    {
      "service_id": "PROCESS_GROUP-XXXX",
      "service_name": "...",
      "requests_per_hour_avg": 0,
      "requests_per_hour_peak": 0,
      "traffic_score": 0.0,
      "internet_facing": false,
      "internet_score": 0.2,
      "ingress_evidence": "publicDomainNames|ingressClass|LoadBalancer|none"
    }
  ],
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator — run before trusting any "absent" finding)

Before using any worker result, scan every WorkerResult for absence claims
("not reached", "no traffic", "owner UNKNOWN with no labels", "not exposed",
etc.). For EACH such claim:

1. Require the worker's **proof** (the unfiltered confirmation query + its
   zero-row result). If the worker did not attach it, the claim is invalid.
2. Run ONE cheap unfiltered confirmation query yourself. Examples:
   - W-reachability "not reached": run
     `fetch spans, from:now()-7d | filter dt.entity.process_group == "{pg_id}" | limit 1`
     — if ANY spans return, the process group IS reachable; the worker mis-scoped.
     Re-dispatch W-reachability for that process group.
   - W-ownership "UNKNOWN": bridge to the serving SERVICE and read its node —
     `fetch spans, from:now()-7d | filter dt.entity.process_group == "{pg_id}" | summarize s = collectDistinct(dt.smartscape.service)`
     then `smartscapeNodes SERVICE | filter id == "{service_id}" | fields properties`
     and scan `properties` for any owner-like label. If found, set owner from it.
3. Only after a confirmation query genuinely returns zero across the broad scope
   may "absent" enter the report.

---

### Phase 2: Prioritization Scoring (Orchestrator, Sequential)

After all workers return (and the Absence Gate has cleared any "absent"
claims), compute a **weight** per affected service:

```
weight = (reachability_score * 0.35)
       + (traffic_score      * 0.25)
       + (internet_score     * 0.20)
       + (error_path_score   * 0.20)
```

Where:
- `reachability_score` (W-reachability): `1.0` if `reached == true`, else `0.0`.
  Optional bonus: `+ 0.1 * (traffic_path_pct / 100)` when code.namespace evidence
  exists, capped at `1.0`.
- `traffic_score` (W-exposure): saturating log10 of peak req/hr, 0.0–1.0.
- `internet_score` (W-exposure): 1.0 if internet-facing, else 0.2.
- `error_path_score` (W-reachability): `min(1.0, error_pct / 10.0)` — caps at
  10% error rate. A high error rate amplifies risk because failure paths often
  pull the vulnerable library (e.g. logging on exception → Log4Shell).

**Sort affected services by `weight` desc.** Top-3 by weight are
`PRIORITY_FIX_LIST`.

**Score interpretation:**
- `>= 0.75` → CRITICAL — fix in next deploy window
- `0.50–0.74` → HIGH — fix this week
- `0.25–0.49` → MEDIUM — fix this sprint
- `< 0.25` → LOW — schedule with regular patching cycle

Per-service decomposition is captured in the Phase 4 Prioritized Fix List table.

---

### Phase 3: Davis CoPilot Synthesis (Orchestrator, Sequential)

Pass the **prioritized fix list + AppSec record** to Davis CoPilot. Do NOT pass
raw worker findings.

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
VULNERABILITY BLAST RADIUS: {ANCHOR_VALUE}
SEVERITY: {SEVERITY} (CVSS {CVSS_SCORE})
AFFECTED SERVICE COUNT: {AFFECTED_COUNT}
TIME WINDOW: {DERIVED_TIME_WINDOW.from} to {DERIVED_TIME_WINDOW.to}

PRIORITIZED FIX LIST (by weighted risk score):

P1 (weight {N}): {service_name} ({owner})
  - Reachability: {reached / traffic_path_pct%}
  - Traffic: {peak_req_per_hr}/hr
  - Internet-facing: {true|false} ({ingress_evidence})
  - Error path involvement: {error_pct}%
  - Affected library versions: [{purl@version}, ...]

P2 (weight {N}): ...
P3 (weight {N}): ...

AFFECTED LIBRARIES: {AFFECTED_LIBRARIES}

QUESTIONS:
1. What is the recommended fixed version for each affected library?
   Cite official advisory (NVD, GitHub Security Advisory, vendor).
2. For the top-3 prioritized services, what's the safest upgrade path
   (semver-major implications, breaking changes, transitive dependencies
   that pin the vulnerable version)?
3. Mitigations applicable BEFORE the upgrade lands (config flags, WAF rules,
   network-level controls)?
4. Validation guidance — what telemetry pattern in Dynatrace would confirm
   a successful remediation?
```

Davis CoPilot response feeds the "Recommended Dependency Upgrade",
"Pre-Upgrade Mitigations", and "Validation Plan" sections.

---

### Phase 4: Report Generation

If `CLEAN_MODE`, build the sanitization map first (identical to dt-rca Phase
1.14 — read that section for full rules) and compose with sanitized names from
the start. Sanitization applies to service names, K8s namespaces, cluster
names, owner team slugs, and the GitHub repo path in PR_REPO. CVE IDs and CVSS
scores are NEVER sanitized — they are public references.

**Emit the report EXACTLY in the section order below — this is the canonical
skeleton, not a menu. Do not reorder, drop, or collapse sections into prose.
Each fact has ONE canonical home (see RULES "De-duplication"); never restate
the same data in another section.**

```markdown
# dt-vuln-blast Vulnerability Blast Radius Report
## {ANCHOR_TYPE}: {ANCHOR_VALUE} — {SEVERITY} (CVSS {CVSS_SCORE})

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Anchor:** {ANCHOR_TYPE} = {ANCHOR_VALUE}  |  **Window:** {from} → {to}  |  **Workers:** W-affected · W-reachability · W-ownership · W-exposure

---

## Executive Summary

### Critical Findings
[One short paragraph. Lead with: CVE / lib + severity, affected service count,
top-1 prioritized service + owner. State the headline risk in ONE sentence.]

### Impact Summary
| Metric | Value | Status |
|--------|-------|--------|
| Affected services | {AFFECTED_COUNT} | {CRITICAL/HIGH/MEDIUM/LOW based on top weight} |
| Reached in production traffic (7d) | {N} | |
| Internet-facing among affected | {N} | |
| Owners identified | {N} / {AFFECTED_COUNT} | {gap callout if any UNKNOWN} |
| Top owner by affected count | {team} ({N} services) | |

### Business Impact
- [Affected services touching customer traffic; total req/hr exposed;
  exploitability if known from Davis synthesis. Business-level bullets, not technical.]

### Immediate Action Required
| Priority | Action | Owner | Urgency |
|----------|--------|-------|---------|
[Top-3 from Prioritized Fix List, with the Davis-recommended fixed version.]

---

## Affected Entity Table
*Source: W-affected — security.events*

| Service | Owner | K8s Workload | Affected Library Versions | State | First Seen |
|---------|-------|--------------|--------------------------|-------|------------|
[ONE row per service in AFFECTED_SERVICES. Sanitized in -clean mode.]

---

## Reachability Evidence
*Source: W-reachability — span-derived call paths*

For EACH service in the Prioritized Fix List (top-3), one subsection:

### {service_name}
- **Reached:** {true|false}  ·  **Traffic path %:** {traffic_path_pct}%
- **Evidence basis:** {span.code.namespace match | entity-library-presence}
- **Entry points (top 5):**
  | Endpoint | Requests (7d) | Failed | Error % |
  |----------|---------------|--------|---------|
- **Error path involvement:** {error_pct}% of root spans failed
- **Outbound DB callouts (if relevant):** {top db.system + operation}

For services outside the top-3, summarize as: `{N} additional services
reached, see Appendix A for full per-service evidence.`

---

## Prioritized Fix List
*Source: Phase 2 weighted scoring*

**Score formula:** `reachability × 0.35 + traffic × 0.25 + internet × 0.20 + error_path × 0.20`.

| Rank | Service | Owner | Weight | Reach | Traffic | Internet | Error Path |
|------|---------|-------|--------|-------|---------|----------|------------|
| P1   | ...     | ...   | 0.82   | 1.00  | 0.75    | 1.00     | 0.45       |
| P2   | ...     | ...   | 0.61   | 1.00  | 0.55    | 0.20     | 0.30       |
| P3   | ...     | ...   | 0.40   | 1.00  | 0.40    | 0.20     | 0.05       |

[One synthesis paragraph: who owns the most P1/P2 services; which cluster /
namespace concentrates the risk; whether internet-facing or internal risk
dominates. Owner-based routing summary lives HERE.]

---

## Recommended Dependency Upgrade
*Source: Davis CoPilot synthesis (Phase 3)*

### Fixed Version
| Vulnerable Library | Fixed Version | Advisory |
|--------------------|---------------|----------|
| {purl@version}     | {fixed_purl}  | {NVD/GHSA link} |

### Upgrade Path
- [Davis CoPilot guidance: semver implications, breaking changes,
  transitive pins, manifest update sketch (no DQL).]

### Pre-Upgrade Mitigations
- [Config flags, WAF rules, network controls — applicable before the
  upgrade lands.]

### Validation Plan
- [Telemetry pattern in Dynatrace that confirms remediation. Reference
  `security.events` re-scan cadence and the expected disappearance of
  the CVE from AFFECTED_ENTITY_IDS.]

---

## Davis Synthesis
*Source: mcp__dynatrace__chat_with_davis_copilot (Phase 3)*

[Full text response from Davis CoPilot, lightly formatted. If Davis was
unavailable, replace this section with the unavailability banner from
"Error Handling" and lean on evidence-based remediation.]

---

## Remediation PR (if --pr provided)

### Target Repository
**Repo:** {PR_REPO}
**Detected language / manifest:** {pom.xml | package.json | requirements.txt | go.mod}
**Branch base:** main (or repo default)

### Drafted Patch
Saved to: `{report_dir}/{cve_or_lib_slug}-upgrade.patch`

```diff
[Unified diff of the manifest change.]
```

### Push Status
- `--apply` NOT provided → **DRAFT ONLY — no PR pushed.** Review the patch,
  then re-run with `--apply` to push.
- `--apply` provided → **PR pushed.** URL: {pr_url} (from
  `mcp__github__create_pull_request` response). Branch:
  `dt-vuln-blast/{cve_slug}`.

*(Omit this entire section if PR_MODE == false.)*

---

## Appendix A: Investigation Details
| Field | Value |
|-------|-------|
| Anchor | {ANCHOR_TYPE}: {ANCHOR_VALUE} |
| CVE(s) | {CVE_IDS joined} |
| CVSS Score | {CVSS_SCORE} |
| Severity | {SEVERITY} |
| Affected Libraries | {AFFECTED_LIBRARIES joined} |
| Affected Service Count | {AFFECTED_COUNT} |
| Workers Dispatched | W-affected, W-reachability, W-ownership, W-exposure |
| Davis CoPilot | {Used / Unavailable} |
| Owners Resolved | {N} / {AFFECTED_COUNT} ({list UNKNOWN services}) |
| PR Mode | {drafted only | pushed | not requested} |

### Full Per-Service Evidence
| Service | Owner | Reach | Traffic/hr (peak) | Internet | Error % | Weight |
|---------|-------|-------|-------------------|----------|---------|--------|
[FULL table — all affected services, not just top-3. This is the single
canonical full list.]

### dt-vuln-blast Worker Telemetry
| Worker | Reference | Key Findings | Gaps |
|--------|-----------|--------------|------|
[W-affected / W-reachability / W-ownership / W-exposure rows]

## Appendix B: Glossary
[Brief — purl, CPE, CVSS, reachability, span.code.namespace, Smartscape, PGI.]

## Appendix C: Query Exchange Log
[One line → "Full query log: {LOG_PATH}". Do NOT inline the DQL.]

---

**Links:**
- [Vulnerability in AppSec](https://{TENANT}.apps.dynatrace.com/ui/apps/dynatrace.security.vulnerabilities/vulnerability/{CVE_ID})
- [NVD Advisory](https://nvd.nist.gov/vuln/detail/{CVE_ID})
- [GitHub Security Advisory](https://github.com/advisories?query={CVE_ID}) (if found)

---

*End of Report*
```

### Pre-Save Self-Check (run before Phase 5 writes anything)

If any item is "no", fix before writing:
- [ ] H1 title + `## {ANCHOR_TYPE}: …` H2 subtitle present
- [ ] Header is `Generated / Analyst: Claude {MODEL} / Environment` (no
      tool-version banner)
- [ ] Executive Summary has all 4 sub-blocks (Critical Findings, Impact
      Summary table, Business Impact bullets, Immediate Action table)
- [ ] Affected Entity Table present with sanitization applied if `-clean`
- [ ] Reachability Evidence section present (top-3 detailed, rest summarized)
- [ ] Prioritized Fix List present (with factor columns inline; no second
      weight table)
- [ ] Recommended Dependency Upgrade present with fixed version table
- [ ] Davis Synthesis present OR replaced with unavailability banner
- [ ] Remediation PR section present only if `PR_MODE == true`
- [ ] Appendix A full per-service table present (single canonical list)
- [ ] No DQL anywhere in the body (Appendix C carries only the log pointer)
- [ ] Footer is `**Links:**` + `*End of Report*` (no tool-version signature)
- [ ] Filename will be `VULN_{ANCHOR_SLUG}_{DATE}[_SANITIZED]`
- [ ] **Output is MD-only by default.** The `.md` is the canonical deliverable
      and is always written. PDF is checked/attempted ONLY if `PDF_MODE == true`,
      and even then is non-fatal (missing engine → one log line, run continues).
- [ ] **Mermaid lint** (if any Mermaid present — typically none in this skill):
      gantt task names have exactly ONE `:`; sequence message text after `:`
      has no `"` and no second `:`; no escaped `\"`; graph/flowchart node
      labels use `<br/>` for line breaks — NEVER literal `\n`.

---

### Phase 5: PR Mode (Optional — only if `PR_MODE == true`)

Runs only when `--pr REPO:"owner/repo"` was provided.

#### Step 5.1 — Locate the repo + detect language

Use `mcp__github__get_file_contents` to probe the repo root for one of these
manifests, in priority order:

| Manifest | Language | Lib coord mapping |
|----------|----------|-------------------|
| `pom.xml` or `build.gradle` / `build.gradle.kts` | Java/JVM | maven purl |
| `package.json` (+ `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml`) | Node.js | npm purl |
| `requirements.txt` / `pyproject.toml` / `Pipfile` | Python | pypi purl |
| `go.mod` | Go | golang purl |

If no manifest is found, abort Phase 5 with a gap noted in the report.

#### Step 5.2 — Compute the upgrade diff

For the detected manifest, change the vulnerable library version from
`AFFECTED_LIBRARIES` to the **fixed version** Davis CoPilot recommended in
Phase 3. Produce a unified diff.

- **Java pom.xml** — locate `<artifactId>{name}</artifactId>` + sibling
  `<version>` and bump.
- **Node package.json** — locate `"{name}": "<range>"` in `dependencies` /
  `devDependencies` and bump to a satisfied range with the fixed version
  (e.g. `^2.17.2`).
- **Python requirements.txt** — locate `{name}==<version>` (or `>=`/`~=`) and
  rewrite to `>={fixed_version},<{next_major}`.
- **Go go.mod** — locate `{module} v<version>` in the `require` block and bump.

Save the patch to `{report_dir}/{cve_or_lib_slug}-upgrade.patch`. Add the patch
path to the Report's "Remediation PR" section.

#### Step 5.3 — Push the PR (only if `--apply` AND `--pr` both present)

**Gate (NON-NEGOTIABLE):** if `APPLY_MODE == true` and `PR_MODE == false`, the
skill already aborted in Phase 0a — this code path is unreachable. Reassert
the gate anyway:

```
if APPLY_MODE and not PR_MODE:
    print("ERROR: --apply requires --pr. Aborting before any GitHub write.")
    exit
```

If both are set:
1. Use `mcp__github__create_branch` to create `dt-vuln-blast/{cve_slug}` off
   the repo default branch.
2. Use `mcp__github__create_or_update_file` to apply the diff to the manifest
   (and to any companion lock file if the language requires it — note this as
   a limitation if the lock file regeneration is non-trivial; for npm/yarn,
   the lock typically needs a human-side `npm install` to refresh integrity
   hashes, so DO NOT attempt to author it).
3. Use `mcp__github__create_pull_request` with:
   - `title`: `chore(security): bump {lib_name} to {fixed_version} ({cve_id})`
   - `body`: short paragraph referencing the dt-vuln-blast report + CVE +
     advisory link + Davis-recommended fixed version. Include the line
     `Generated by dt-vuln-blast.` No customer telemetry in the body.
   - `head`: `dt-vuln-blast/{cve_slug}`
   - `base`: repo default branch (queried via `mcp__github__list_branches` or
     repo metadata).
   - `draft`: `true` — always create as a draft. Owners take it out of draft
     after their own verification.
4. Capture `pr_url` from the response and write it into the report's
   "Push Status" subsection.

If `--apply` was NOT provided, log to Phase 5 telemetry:
`"PR drafted only — patch at {patch_path}; re-run with --apply to push."` and
STOP. Never implicit writes.

---

### Phase 6: Save Report (Markdown)

Run the Pre-Save Self-Check (end of Phase 4) before writing anything.

**Default: write the `.md` ONLY.** The Markdown is the canonical deliverable.
Always write it; never gate it on anything.

**PDF is opt-in and non-fatal.** Only if `PDF_MODE == true`: attempt the
inherited dt-rca PDF path (filename generation, the `md-to-pdf` command, the
intermediate `_pdf.md` Mermaid→ASCII strategy, and cleanup — read
`~/.claude/skills/dt-rca/SKILL.md` Phases 3+4 for the mechanics). **Wrap the PDF
render so a missing/failed engine logs exactly one line and the run continues:**
```bash
# Only runs when PDF_MODE == true
if ! command -v md-to-pdf >/dev/null 2>&1; then
  { printf '[%s] PDF engine unavailable — Markdown only\n' "$(date -u +%FT%TZ)" >> "$LOG"; } &
else
  md-to-pdf "$PDF_INTERMEDIATE" 2>>"$LOG" \
    || { printf '[%s] PDF render failed — Markdown only\n' "$(date -u +%FT%TZ)" >> "$LOG"; }
fi
```
NEVER hard-fail a run on PDF. Do NOT delete the inherited Mermaid/ASCII/diagram
rules — they still apply when a PDF is requested.

NEVER use the dt-rca or dt-rcf filename forms — dt-vuln-blast reports are
`VULN_*`. **This skill overrides dt-rca's "always generate PDF" behavior
locally — dt-rca / dt-rcf themselves are never edited.**

Filenames:
```
Normal:   VULN_{ANCHOR_SLUG}_{DATE}.md  /  VULN_{ANCHOR_SLUG}_{DATE}.pdf
Clean:    VULN_{ANCHOR_SLUG}_{DATE}_SANITIZED.md  /  ...SANITIZED.pdf

ANCHOR_SLUG examples:
  CVE-2021-44228                              → CVE-2021-44228
  LIB:"org.apache.logging.log4j:log4j-core@2.14.1" → log4j-core_2_14_1
  LIB:"lodash@4.17.20"                         → lodash_4_17_20
```

---

### Phase 7: Output Summary

```markdown
## dt-vuln-blast Report Generated

| Format | Filename |
|--------|----------|
| Markdown | VULN_{SLUG}.md |
| PDF | VULN_{SLUG}.pdf (or "skipped (MD-only default; pass --pdf to enable)") |
| Log | VULN_{SLUG}.log |
| Patch (if --pr) | {SLUG}-upgrade.patch |

**Anchor:** {ANCHOR_TYPE}: {ANCHOR_VALUE}
**Severity:** {SEVERITY} (CVSS {CVSS_SCORE})
**Affected Services:** {AFFECTED_COUNT}
**Time Window:** {from} → {to}

### Prioritized Fix List
| Rank | Service | Owner | Weight |
| P1   | ...     | ...   | {N}    |
| P2   | ...     | ...   | {N}    |
| P3   | ...     | ...   | {N}    |

### dt-vuln-blast Telemetry
| Worker | Reference | Findings | Gaps |
| W-affected     | dt-obs-services (entity)  | ... | ... |
| W-reachability | dt-obs-tracing/failure+http+db | ... | ... |
| W-ownership    | dt-obs-kubernetes/labels  | ... | ... |
| W-exposure     | dt-obs-services/service-metrics | ... | ... |

### PR Status
- **Drafted:** {yes/no — patch at {path}}
- **Pushed:** {yes/no — URL: {pr_url} or "--apply not provided"}
```

If `CLEAN_MODE`, append sanitization key to console (never to file) per dt-rca
rules.

---

## RULES (NON-NEGOTIABLE)

### Inherited from dt-rca + dt-rcf

Rules 1–28 from `~/.claude/skills/dt-rcf/SKILL.md` apply unchanged. They cover:
Smartscape-native entity model, sub-skills as DQL authority, no skipped data
collection, Mermaid-in-MD / ASCII-in-PDF, executive audience, Appendix query
logging, historical verification, compact diagrams, two-file PDF strategy,
clean-mode zero-leaks / consistency / plausibility, the DQL backtick /
timestamp / count-alias / canonical-link rules, and parallel worker dispatch
in a single message.

### New in dt-vuln-blast — vulnerability domain

29. **Library coordinates MUST normalize to purl** before AppSec lookup. Raw
    `LIB:"name@version"` without purl-normalization is invalid. The
    normalization table in Phase 0b is the canonical authority.

30. **AppSec fields are PROBE-BOUND, never hardcoded.** The Phase 0b.0
    Substrate Probe discovers `event.kind`, the CVE field, and the
    affected-entity field at runtime from a candidate list ordered
    `[corrected-2026-06-03, legacy]`, and the lookup queries interpolate the
    bound values. Validated PRIMARY values (tenant `demo`, 2026-06-03):
    `event.kind=="SECURITY_EVENT"`, CVE via
    `in("{CVE_ID}", vulnerability.references.cve)`, affected entities in
    `affected_entity.id` (values are **`PROCESS_GROUP-*`**). The four workers
    therefore scope off PROCESS_GROUP via the standard span dimension
    `dt.entity.process_group == "{pg_id}"` (NOT `smartscapeNodes PROCESS_GROUP`,
    which is 0 nodes, and NOT `dt.smartscape.process_group`, which is NULL on
    spans). PG→SERVICE enrichment (span → `dt.smartscape.service`) is optional
    and best-effort — it never gates a run.

31. **Reachability is binary + bonus, not a gradient.** A service is `reached`
    (1.0) or `not reached` (0.0) based on root-span presence in the window;
    the `traffic_path_pct` is a bonus signal, not the score. This prevents
    "kind of reachable" middle scores that confuse remediation prioritization.

32. **Owner UNKNOWN is a gap, not a default.** Any service with
    `owner == UNKNOWN` MUST be flagged in the Impact Summary and Appendix A
    as a routing gap. Never assign a placeholder owner.

33. **Score weights are fixed.** `0.35 reachability + 0.25 traffic +
    0.20 internet + 0.20 error_path`. Do NOT adjust per-run. If the weighting
    becomes inadequate, fix it here and re-validate — never silently in code.

34. **`--apply` requires `--pr` — never implicit writes.** Phase 0a aborts if
    `--apply` appears without `--pr`. Phase 5.3 reasserts the gate before any
    GitHub MCP write call. Drafted patches NEVER push without explicit
    `--apply`.

35. **PRs ALWAYS created as draft.** Owners take them out of draft after their
    own verification. `mcp__github__create_pull_request` is called with
    `draft: true` — non-negotiable.

36. **Davis CoPilot is the authority for fixed versions.** Do NOT guess
    "the next semver" or extrapolate from NVD ranges. The recommended fixed
    version comes from Davis synthesis in Phase 3. If Davis is unavailable,
    the report says "fixed version pending Davis CoPilot — defer manifest
    bump" and the patch is NOT generated.

37. **Filename = `VULN_{ANCHOR_SLUG}_{DATE}[_SANITIZED]`** — NEVER `RCF_*` or
    `Problem_{ID}_Analysis_Report`.

### Data integrity (inherited applied)

38. **A worker's empty/error result is never a conclusion.** Workers attach
    proof for any absence claim; the orchestrator runs the Phase 1.5 Absence
    Gate before any "not reached / no owner / not exposed" enters the report.
    Scope runtime metrics by `dt.entity.process_group_instance`, never the
    empty `dt.process_group.id`.

39. **dtctl auth: on-disk tokens, lazy login, orchestrator-only.** Tokens
    are stored on disk (`DTCTL_TOKEN_STORAGE=file`, harness env — never the
    Keychain) and auto-refresh. Workers NEVER log in. On `<gap: auth>` the
    orchestrator refreshes once and re-dispatches the failing worker. Login is
    interactive-and-orchestrator-only; workers never log in or refresh, and the
    orchestrator never races concurrent/non-interactive logins (see Auth
    Hardening 2026-06-03).

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section "PDF
DIAGRAM RULES". Summary: no Mermaid in PDFs — ASCII art only, inside plain
code blocks, 80-char max width. (Note: dt-vuln-blast reports typically have
NO diagrams — the data is tabular. If a Mermaid block is added for owner
topology or fix-list sankey, the same rules apply.)

---

## MERMAID SYNTAX RULES

Inherits dt-rca's "MERMAID SYNTAX RULES" + dt-rcf's label-sanitization rules
(no `:` in gantt task names; no literal double-quotes in sequence message
labels; no `\n` in graph node labels — use `<br/>`).

---

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"DIAGRAM SIZING RULES".

---

## ERROR HANDLING

### No affected entities found
```markdown
## No Affected Entities

Anchor `{ANCHOR_TYPE}: {ANCHOR_VALUE}` matched zero entities in
`security.events` over the last 30 days.

**Possible causes:**
- CVE not yet detected in this tenant's AppSec scan window
- Library version string does not match the purl normalization (re-run with
  full purl form: `LIB:"pkg:maven/group/name@version"`)
- Affected entities were resolved/ignored and filtered out of AppSec
- AppSec module not enabled for the tenant

**Try:**
- Re-run with the explicit purl form
- Check the AppSec UI for the CVE
- Confirm AppSec is enabled: `dtctl query 'fetch security.events | limit 1'`
```

### Worker fails entirely
Orchestrator marks the corresponding section with `⚠️ gap`, notes the worker
+ reason in Appendix A, and continues. Does NOT auto-retry beyond the single
re-dispatch budgeted in Phase 1 — a third worker call would likely fail for
the same reason and serialize the run.

### Davis CoPilot unavailable
Generate report from Phase 2 scoring alone. Add banner:
```
> **Note:** Davis CoPilot synthesis unavailable. Fixed-version recommendation
> is deferred — review NVD / GHSA manually before applying any upgrade. PR
> mode (--pr) is DISABLED for this run; re-run with Davis available to
> generate the patch.
```
**When Davis is unavailable, Phase 5 (PR Mode) is SKIPPED entirely.** The
report explains this in the "Remediation PR" section.

### `--apply` without `--pr`
```markdown
## Error: --apply Requires --pr

`--apply` instructs dt-vuln-blast to push a remediation PR to GitHub.
Without `--pr REPO:"owner/repo"`, there is no repo to push to and no patch
to apply.

**Try:**
  /dt-vuln-blast {ANCHOR_VALUE} --pr REPO:"owner/repo" --apply

Aborting before any write.
```

### Repo not found / manifest not detected (Phase 5)
Drop a `⚠️ gap` row into the Remediation PR section. Do NOT exit — the
report is still useful for the prioritized fix list and owner routing.

---

## QUALITY RULES

These are run-time checks the orchestrator validates before saving the report.
If any fails, fix before Phase 6.

1. **Anchor parsed unambiguously** — exactly ONE of `CVE` or `LIBRARY`.
2. **AppSec record fetched** — `security.events` returned at least one
   row for the anchor; CVE_IDS, SEVERITY, CVSS_SCORE, AFFECTED_LIBRARIES are
   populated.
3. **AFFECTED_COUNT > 0** — otherwise the "No Affected Entities" error
   report fires instead.
4. **All four workers returned** (or were re-dispatched once) — gaps recorded
   in Appendix A if any failed twice.
5. **Absence Gate cleared** — no "not reached / not exposed / UNKNOWN owner"
   claim entered the report without a confirmation query.
6. **Score weights sum to 1.00** — `0.35 + 0.25 + 0.20 + 0.20 == 1.00`.
7. **Per-service evidence completeness** — top-3 services have entry_points,
   error_pct, traffic_score, internet_score, owner all populated. UNKNOWN
   owners are flagged, not silently omitted.
8. **PR gating respected** — Phase 5 ran only if `PR_MODE == true`; Step 5.3
   ran only if `APPLY_MODE == true AND PR_MODE == true`; PR was created as
   `draft: true`.
9. **Sanitization consistency** — if `CLEAN_MODE`, no service name, K8s
   namespace, cluster name, or owner team slug appears in unsanitized form
   anywhere in the report (CVE IDs and CVSS scores excluded — public refs).
10. **Filename + appendix discipline** — file is `VULN_*`, never `RCF_*` /
    `Problem_*`. No DQL in body — only in Appendix C log pointer.

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously until
the report is saved (and the PR is drafted / pushed if requested).

**Do not stop for confirmation between phases. Do not ask questions. Generate
the complete report. Never push a PR unless BOTH `--pr` and `--apply` are
present.**
