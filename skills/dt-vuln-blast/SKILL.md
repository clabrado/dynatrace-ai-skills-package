---
name: dt-vuln-blast
description: >
  Vulnerability Blast Radius — agentic, multi-worker analysis that turns a CVE or
  library coordinate into a prioritized list of impacted production services with
  reachability evidence, traffic + exposure weighting, ownership routing, and an
  optional draft remediation PR. Anchor types: CVE-YYYY-NNNN or LIB:"name@version".
  Weighted scoring formula: reachability 0.35 / traffic 0.25 / internet 0.20 / error-path 0.20.
---

# dt-vuln-blast — Vulnerability Blast Radius

Agentic vulnerability impact-analysis skill. Accepts a CVE or library coordinate,
dispatches parallel workers, scores each affected service by reachability, traffic,
internet-exposure, and error-path involvement, and produces a prioritized fix list
(Markdown + PDF). Optional --pr / --apply mode drafts and pushes a dependency
upgrade PR via the GitHub MCP.

## Usage

```
/dt-vuln-blast CVE-2021-44228                                    # CVE anchor (Log4Shell)
/dt-vuln-blast LIB:"org.apache.logging.log4j:log4j-core@2.14.1"  # Library coord anchor
/dt-vuln-blast CVE-2022-22965 -clean                             # Sanitized report
/dt-vuln-blast CVE-2021-44228 --pr REPO:"acme/payment-api"       # Draft remediation PR (no push)
/dt-vuln-blast CVE-2021-44228 --pr REPO:"acme/payment-api" --apply  # Draft AND push the PR
```

## Arguments

| Arg / flag | Description |
|---|---|
| `CVE-YYYY-NNNN` | Direct CVE lookup against `security.events` |
| `LIB:"name@version"` | Library coordinate; normalized to purl before lookup |
| `-clean` | Sanitize service / namespace / cluster / owner / tenant URL |
| `--pr REPO:"owner/repo"` | Draft upgrade patch (saves `{cve}-upgrade.patch`; no push) |
| `--apply` | Push the drafted PR — **requires `--pr`**; never implicit |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with AppSec enabled (`security.events` data)
- dynatrace-for-ai skills: `dt-obs-services`, `dt-obs-tracing`, `dt-obs-kubernetes`
- `md-to-pdf` for PDF output
- GitHub MCP (`mcp__github__*`) — **only** for `--pr` / `--apply` mode

## Anchor Types

| Anchor | Recognizer | Example |
|---|---|---|
| CVE | `CVE-\d{4}-\d+` | `CVE-2021-44228` |
| LIBRARY | `LIB:"<coord>"` | `LIB:"log4j-core@2.14.1"`, `LIB:"lodash@4.17.20"` |

**Library coordinate normalization to purl:**
- `group:name@version` (contains `:`) → `pkg:maven/<group>/<name>@<version>`
- `name@version` (no `:`) → `pkg:npm/<name>@<version>` (primary) or `pkg:pypi/<name>@<version>`
- Already starts with `pkg:` → pass through

## Sub-Skills Loaded Per Phase

| Phase | Reference file (read ONCE at worker start) |
|---|---|
| W-affected     | `dt-obs-services/SKILL.md` |
| W-reachability | `dt-obs-tracing/references/failure-detection.md` + `http-spans.md` / `database-spans.md` |
| W-ownership    | `dt-obs-kubernetes/references/labels-annotations.md` + `workload-health.md` |
| W-exposure     | `dt-obs-services/references/service-metrics.md` |

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

**`--apply` gate:** if `APPLY_MODE == true` and `PR_MODE == false` → print error and stop immediately.

### Phase 0b: AppSec Bootstrap (Sequential, 2 queries)

**AppSec data source: `security.events` (no `dt.` prefix) — validated 2026-05-28.**

Step 1: Normalize LIBRARY coordinate to purl (skip if CVE anchor).

Step 2: Lookup affected entities:
```dql
-- CVE anchor
fetch security.events, from:now()-30d
| filter event.kind == "VULNERABILITY_STATE_REPORT_EVENT"
| filter vulnerability.cve.id == "{CVE_ID}"
| filter isNotNull(vulnerability.id)  -- REQUIRED: drops records without vuln.id
| summarize takeFirst(vulnerability.severity) as severity,
            takeFirst(vulnerability.cvss.score) as cvss_score,
            collectDistinct(vulnerability.affected_library.purl) as affected_libraries,
            collectDistinct(coalesce(smartscape.affected_entity.ids, affected_entity_ids)) as affected_entity_ids
```

If `AFFECTED_COUNT == 0` → emit "No Affected Entities" error and exit.

Resolve entity names:
```dql
smartscapeNodes SERVICE | filter id in {AFFECTED_ENTITY_IDS_ARRAY} | fields id, name, properties
```

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

**W-affected:** Per-service affected component breakdown, vulnerability lifecycle status, first/last seen timestamps from `security.events`.

**W-reachability:** For each affected service:
- HTTP/RPC entry points (root spans)
- Error-path involvement (error_pct)
- Outbound DB callouts (if DB/ORM library)
- Reachability evidence: `reached = true` if root-span traffic > 0 in 7d window

**W-ownership:** Resolve K8s workload → owner labels/annotations.
Owner label precedence: `dynatrace.tag.owner` → `app.kubernetes.io/owner` → `owner` label → `team` annotation/label → `cost-center`.
`owner = "UNKNOWN"` if none found — flagged as a routing gap.

**W-exposure:** Traffic volume (req/hr avg + peak), internet-facing classification via Smartscape `publicDomainNames` or K8s LoadBalancer/IngressClass.

### Phase 1.5: Absence Gate

For "not reached / UNKNOWN owner / not exposed" claims: require worker proof + run one unfiltered confirmation.

### Phase 2: Prioritization Scoring

```
weight = (reachability_score * 0.35)
       + (traffic_score      * 0.25)
       + (internet_score     * 0.20)
       + (error_path_score   * 0.20)
```

| Factor | Score formula |
|---|---|
| `reachability_score` | 1.0 if `reached == true`, else 0.0. Bonus: +0.1 × (traffic_path_pct/100) capped at 1.0 |
| `traffic_score` | `min(1.0, log10(1 + requests_per_hour_peak) / 4.0)` (saturates near 10K req/h) |
| `internet_score` | 1.0 if internet-facing, else 0.2 |
| `error_path_score` | `min(1.0, error_pct / 10.0)` (caps at 10% error rate) |

Score thresholds:
- ≥ 0.75 → CRITICAL — fix in next deploy window
- 0.50-0.74 → HIGH — fix this week
- 0.25-0.49 → MEDIUM — fix this sprint
- < 0.25 → LOW — regular patching cycle

### Phase 3: Davis CoPilot Synthesis

Pass prioritized fix list + AppSec record to Davis. Ask for: recommended fixed version (with advisory citation), safest upgrade path, pre-upgrade mitigations, validation telemetry pattern.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 + H2 with CVE/LIB anchor + SEVERITY + CVSS
2. Header: Generated / Analyst / Environment / Anchor / Window / Workers
3. Executive Summary (Critical Findings, Impact Summary, Business Impact, Immediate Action)
4. Affected Entity Table (all services, one row each)
5. Reachability Evidence (top-3 detailed, rest summarized)
6. Prioritized Fix List (factor columns + synthesis paragraph)
7. Recommended Dependency Upgrade (Fixed Version table + Upgrade Path + Pre-Upgrade Mitigations + Validation Plan)
8. Davis Synthesis
9. Remediation PR section (only if PR_MODE)
10. Appendix A: Full per-service evidence table + worker telemetry
11. Links (NVD advisory + GitHub Security Advisory) + *End of Report*

**Filenames:**
```
Normal: VULN_{ANCHOR_SLUG}_{DATE}.md / .pdf
Clean:  VULN_{ANCHOR_SLUG}_{DATE}_SANITIZED.md / .pdf
```

### Phase 5: PR Mode (only if --pr)

Step 5.1: Probe repo root for manifest (`pom.xml`, `package.json`, `requirements.txt`, `go.mod`).
Step 5.2: Compute upgrade diff (bump vulnerable library to Davis-recommended fixed version).
Step 5.3: Only if `--apply` AND `--pr`:
- `mcp__github__create_branch` → `dt-vuln-blast/{cve_slug}`
- `mcp__github__create_or_update_file` → apply diff
- `mcp__github__create_pull_request` with `draft: true` (ALWAYS draft)

**PRs are ALWAYS created as draft.** `--apply` without `--pr` → error and stop.

## QUALITY RULES (NON-NEGOTIABLE)

1. **Library coordinates normalize to purl** before AppSec lookup.
2. **AppSec dual-model lookup** — always `coalesce(smartscape.affected_entity.ids, affected_entity_ids)`.
3. **`filter isNotNull(vulnerability.id)` required** on `security.events` aggregations.
4. **Reachability is binary + bonus** — reached (1.0) or not (0.0); traffic_path_pct is bonus only.
5. **UNKNOWN owner is a gap** — always flagged, never silently omitted.
6. **Score weights are fixed** — 0.35/0.25/0.20/0.20. Never adjust per-run.
7. **`--apply` requires `--pr`** — gate enforced at parse time AND pre-write. Never implicit writes.
8. **PRs always `draft: true`** — owners take them out of draft after verification.
9. **Davis CoPilot is authority for fixed versions** — never guess the next semver.
10. **If Davis unavailable, skip PR generation** entirely.

## Real-World Use Cases

1. **Incoming CVE triage at 8am** — security team posts CVE; SE runs skill to show which services actually reach the vulnerable code, sorted by exploitability.
2. **Log4Shell-class supply-chain incident** — pull blast radius across multi-tenant K8s ownership tags to coordinate fan-out.
3. **Sprint planning by reachability** — drop top N vulns through skill; teams prioritize by reachability score, not CVSS alone.
4. **Customer-facing AppSec demo** — `-clean` mode produces shareable artifact with no customer entity names.
5. **Automated remediation PR** — high-confidence single-dependency CVEs flow through `--pr` for owning team to review.
