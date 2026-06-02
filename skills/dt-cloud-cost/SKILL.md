---
name: dt-cloud-cost
description: >
  Cloud Cost Attribution (FinOps) for Dynatrace — agentic, multi-cloud (AWS,
  Azure, GCP) cost rollup by tag/label with delta analysis vs prior period.
  Anchored on CLOUD + optional SCOPE (host or region) + TAG, dispatches
  parallel cost workers against Grail bizevents emitted by the Dynatrace
  carbon-impact app, and produces a polished MD+PDF FinOps report. Validated
  substrate: bizevents with event.provider == "dynatrace.biz.carbon".
---

# dt-cloud-cost — Cloud Cost Attribution (FinOps)

Agentic, multi-cloud cost attribution skill. Parallel cost workers backed by
`dt-obs-{aws|azure|gcp}` references + Grail bizevents from the Dynatrace
carbon-impact app. Produces a complete FinOps attribution report (MD+PDF)
showing who owns the spend, what changed period-over-period, which initiatives
are new, and which workloads have been decommissioned.

**Substrate:** v1 reads bizevents with `event.provider == "dynatrace.biz.carbon"`. There is NO `fetch billing` and NO `billing.cost.usd`. Validated 2026-05: Azure $152K / AWS $55K over 3 days.

## Usage

```
/dt-cloud-cost CLOUD:aws TAG:CostCenter
/dt-cloud-cost CLOUD:azure TAG:dt_owner_team SCOPE:westeurope
/dt-cloud-cost CLOUD:gcp TAG:team SCOPE:us-central1 -clean
/dt-cloud-cost CLOUD:aws TAG:Project --list-price --review
```

## Arguments

| Arg / flag | Description |
|---|---|
| `CLOUD:<aws\|azure\|gcp>` | REQUIRED. Provider filter. |
| `TAG:<key>` | REQUIRED. Attribution key (resolved via Smartscape HOST `tags[]` join on `dt.entity.host`). |
| `SCOPE:<region\|HOST-id>` | OPTIONAL. Region (e.g. `us-east-1`, `westeurope`) or `HOST-XXXXXXXX`. |
| `--list-price` | Use `cost.list.price` / `price.total` instead of `cost.list.spend` / `cost.total` (catalog vs actual). |
| `-clean` | Sanitize identifying names. |
| `--review` | Add reviewer subagent quality gate. |
| `PERIOD:<YYYY-MM>` | Override current period (defaults to current calendar month). |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with carbon-impact app ingesting for the target cloud provider
- dynatrace-for-ai skills: `dt-obs-aws` / `dt-obs-azure` / `dt-obs-gcp` (cost-optimization + resource-ownership refs)
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

| Phase | Cloud | Reference file (read ONCE at worker start) |
|---|---|---|
| W-cost | aws | `dt-obs-aws/references/resource-ownership.md` + `cost-optimization.md` |
| W-cost | azure | `dt-obs-azure/references/resource-ownership.md` + `cost-optimization.md` |
| W-cost | gcp | `dt-obs-gcp/references/resource-ownership.md` + `resource-management.md` |
| W-baseline | (same cloud) | Same files; different time window |
| W-movers | (same cloud) | Joins W-cost and W-baseline results |
| W-attribution | (same cloud) | + entity-specific service queries |

## ENTITY MODEL — CARBON-APP BIZEVENTS + SMARTSCAPE HOST JOIN

v1 reads two substrates:
1. **Cost numbers** from `fetch bizevents | filter event.provider == "dynatrace.biz.carbon"`. NO `classicEntitySelector()`. NO `fetch billing`.
2. **Tag values** from `smartscapeNodes "HOST"` — bizevents carry `dt.entity.host` but NOT `tags[]`. Tag attribution = join.

**Validated bizevent schema:**

| Field | Populated on event.type | Notes |
|---|---|---|
| `event.provider` | all | always `"dynatrace.biz.carbon"` |
| `event.type` | all | `cost.list.spend` (default) or `cost.list.price` |
| `cost.total` | `cost.list.spend` | USD spend |
| `price.total` | `cost.list.price` | USD catalog price |
| `cloud.provider` | all | `aws` / `azure` / `gcp` |
| `cloud.region` | all | e.g. `us-east-1` |
| `resource.category` | all | e.g. `instance`, `storage` |
| `dt.entity.host` | all | join key to Smartscape |

Carbon-emission events (`event.type` starts with `carbon.measurement`) carry emissions, NOT USD. Must be excluded.

**Tag attribution pattern (Smartscape HOST join):**
```dql
fetch bizevents, from:toTimestamp("{PERIOD_START}"), to:toTimestamp("{PERIOD_END}")
| filter event.provider == "dynatrace.biz.carbon"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
| lookup [
    smartscapeNodes "HOST"
    | filter isNotNull(tags[`{TAG_KEY}`])
    | fields id, tag_value = tags[`{TAG_KEY}`]
  ], sourceField: dt.entity.host, lookupField: id, prefix: "host_"
| filter isNotNull(host_tag_value)
| summarize cost_usd = sum({COST_FIELD}), by: { tag_value = host_tag_value }
| sort cost_usd desc | limit 20
```

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0b: Substrate Validation + Period Bootstrap

1. Classify SCOPE (HOST-id vs region vs none)
2. Confirm carbon-app ingestion is live for the CLOUD:
```dql
fetch bizevents, from: now() - 7d
| filter event.provider == "dynatrace.biz.carbon" and cloud.provider == "{CLOUD}"
| summarize event_count = count(), distinct_hosts = countDistinct(dt.entity.host)
```
If `event_count == 0` → exit with "carbon-app not ingesting for CLOUD" error.

3. Derive current + prior period bounds (ISO-8601).

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

**W-cost:** Current period spend by TAG value (top 20) + untagged spend + total + dimensional rollup.

**W-baseline:** Prior period spend by TAG value (same queries, different window).

**W-movers:** ONE joined query spanning both periods:
```dql
fetch bizevents, from:toTimestamp("{PRIOR_START}"), to:toTimestamp("{CURRENT_END}")
| filter ...
| fieldsAdd period = if(timestamp < toTimestamp("{CURRENT_START}"), "prior", else: "current")
| summarize cost_usd = sum({COST_FIELD}), by: { tag_value = host_tag_value, period }
```
Worker pivots, computes delta, returns top 10 movers + new/dropped tag values.

**W-attribution:** For top 5 tag values, resource-category + instance-type breakdown + top 5 hosts by spend.

### Phase 1.5: Billing-Empty Gate (HARD FAIL)

If `event_count == 0` for current period after workers return → FAIL FAST. Never produce a $0 report silently.

### Phase 2: Attribution Synthesis

Answers four questions:
1. **Who owns the spend?** Top 5 tag values
2. **What changed?** Top movers (positive AND negative)
3. **What's new?** New tag values (new initiatives)
4. **What dropped off?** Dropped tag values (decommissioned workloads)

### Phase 3: Davis CoPilot Synthesis (Conditional)

Invoke Davis CoPilot ONLY when:
- `total_delta_pct` exceeds ±15%
- `untagged_pct_current > 20%`
- ≥3 positive movers each exceeding 20% growth
- A dropped tag value previously accounted for >10% of total spend

Skip for flat steady-state reports.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 + H2 with CLOUD / SCOPE / TAG / PERIOD
2. Header: Generated / Analyst / Environment / Cloud / Scope / Attribution Key / Periods
3. Executive Summary (Headline Metrics, Narrative, Immediate Action, Spend Trend chart)
4. Cost Attribution Table (canonical TAG × spend × delta table)
5. Top Movers (Positive + Negative subsections)
6. Per-Owner Resource Detail (top 5 owners)
7. New & Dropped Initiatives
8. Recommended Optimizations
9. Appendix A: Methodology + constraints + worker telemetry
10. Links + *End of Report*

**Filenames:**
```
Normal: Cost_{CLOUD}_{SCOPE}_{PERIOD}_Report.md / .pdf
Clean:  Cost_{CLOUD}_{SCOPE}_{PERIOD}_Report_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **`fetch billing` does NOT exist** — source is `bizevents` with `event.provider == "dynatrace.biz.carbon"`.
2. **No native cloud account IDs in SCOPE (v1)** — SCOPE accepts `cloud.region` or `HOST-id` only.
3. **Schema discovery over guessing** — if `FIELD_DOES_NOT_EXIST` error, run one probe and record in `field_discovery`.
4. **Carbon-emission events excluded** — `event.type` starting with `carbon.measurement` carries emissions, NOT USD.
5. **Billing-Empty Gate is non-negotiable** — never produce a $0 report; fail fast with carbon-app onboarding instructions.
6. **Tagged + untagged must sum to total** — within $1 tolerance.
7. **All four workers dispatch in ONE message** — sequential dispatch defeats the architecture.
8. **Davis CoPilot is optional** — invoke only when data is rich (see triggers above).
9. **Filename = `Cost_*`** — never dt-rca or dt-rcf naming.

## Real-World Use Cases

1. **Monthly FinOps review** — pull all three clouds, attribute by team tag, deliver to leadership.
2. **Budget anomaly investigation** — a team blew their budget; this report shows top movers + new initiatives.
3. **Decommission opportunity finder** — `--list-price` against unused inventory surfaces over-provisioned resources.
4. **Customer FinOps demo** — `-clean` shareable cost attribution artifact.
5. **Tag-hygiene audit** — untagged spend surfaces what teams haven't labeled their resources.

## Substrate Notes (Validated 2026-05-28)

- `fetch billing` confirmed NOT to exist
- Cost fields confirmed: `cost.total` (spend) and `price.total` (catalog list)
- Validated populated: Azure $152K / AWS $55K over 3-day validation window
- Host tag join pattern confirmed working
