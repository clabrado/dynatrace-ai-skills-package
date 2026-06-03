---
name: dt-cloud-cost
description: >-
  Cloud Cost Attribution (FinOps) for Dynatrace — agentic, multi-cloud (AWS,
  Azure, GCP) cost rollup by tag/label with delta analysis vs prior period.
  Anchored on CLOUD + optional SCOPE (host or region) + TAG, dispatches
  parallel cost workers against Grail bizevents emitted by the Dynatrace
  carbon-impact app (`event.provider == "dynatrace.biz.carbon"`), builds an
  attribution narrative (who owns the spend, what changed, what's new, what
  dropped off), optionally synthesizes via Davis CoPilot, and produces a
  polished Markdown FinOps report (PDF optional via `--pdf`). Use for monthly
  cost reviews, chargeback/showback analysis, budget anomaly investigation,
  dimensional rollups (region / resource-category / instance-type) and tag-based
  ownership rollups (via Smartscape HOST tag join, when hosts are tagged), and
  identifying new initiatives or decommissioned workloads. Triggers: cost attribution,
  chargeback, showback, FinOps report, cloud spend, AWS cost by tag, Azure
  spend by region, GCP spend by host, top cost movers, cost delta,
  budget anomaly, untagged spend.
---

# dt-cloud-cost — Cloud Cost Attribution (FinOps)

Agentic, multi-cloud cost attribution skill. Accepts a CLOUD + optional SCOPE +
TAG anchor, dispatches parallel cost workers backed by the maintained
`dt-obs-{aws|azure|gcp}` sub-skills + Grail bizevents from the Dynatrace
carbon-impact app, and produces a complete FinOps attribution report
(**Markdown by default; PDF opt-in via `--pdf`**) showing who owns the spend,
what changed period-over-period, which initiatives are new, and which workloads
have been decommissioned. Attribution defaults to **native event dimensions**
(`cloud.region` / `resource.category` / `resource.instance.type`); tag-based
ownership (Smartscape HOST `tags[]` join) is an OPTIONAL overlay that engages
only when a `TAG:` is supplied AND the host join resolves.

**Substrate (tenant-validated 2026-05; re-validated 2026-06-03):** v1 reads
bizevents emitted by the Dynatrace carbon-impact app — `fetch bizevents | filter
event.provider == "dynatrace.biz.carbon"`. Two USD-bearing event types exist:
- `cost.list.price` → field `price.total` (currency `price.currency`) — the
  populated, authoritative cost feed on the validated tenant
  (**33,963 events / USD 152,091 Azure / 3d**). **This is the DEFAULT** as of
  2026-06-03 (CC-1).
- `cost.list.spend` → field `cost.total` (currency `cost.currency`) — realized
  spend. On the validated tenant this feed is **nearly empty** (361 events /
  USD 68 Azure / 3d), so it is NOT the default. Opt in via `--spend` if your tenant
  populates it. Legacy `--list-price` is a no-op alias (price is already default).

> The default is chosen at runtime by the **Phase 0b.0 Substrate Probe** — it
> counts rows per `event.type` and binds whichever USD-bearing type is populated.
> The static default above is only the probe's PRIMARY candidate.

There is NO `fetch billing` data object on Grail and NO `billing.cost.usd`
field. Other billing-event ingestion shapes (direct AWS CUR exports, Azure
Cost Management exports, GCP Cloud Billing exports) are NOT supported by v1
and require their own extractor.

**Attribution model (CC-2, tenant-validated 2026-06-03):** cost events carry
`dt.entity.host` as a SCHEMA field but it is **NULL on every cost event** on the
validated tenant (`with_host=0` across all 33,963 Azure price events). The host→tag
join therefore yields nothing. The PRIMARY attribution axis is the **native event
dimensions** `cloud.region` / `resource.category` / `resource.instance.type`
(these are always populated). Tag attribution via the Smartscape HOST `tags[]`
join is an OPTIONAL overlay that engages only when `TAG:` is supplied AND the host
join resolves to ≥1 row; otherwise the skill renders the dimensional report and
notes the tag overlay was unavailable.

Carbon-emission events (`event.type` starts with `carbon.measurement`) carry
emissions data, not USD, and are explicitly ignored by this skill.

This skill is a Tier-2 derivative of `dt-rcf` — same execution substrate (parallel
workers, reference-driven DQL, orchestrator-only auth), different domain. Output
is **Markdown by default; PDF is opt-in via `--pdf`** (Mandate 1).

---

## Usage

### Dimensional (no TAG — the safe default)

```
/dt-cloud-cost CLOUD:azure                         # dimensional rollup, no tag join
/dt-cloud-cost CLOUD:aws SCOPE:us-east-1           # region-scoped dimensional rollup
/dt-cloud-cost CLOUD:gcp --pdf                     # dimensional + opt-in PDF
```

`TAG:` is OPTIONAL. With no TAG, the report attributes spend by native event
dimensions (`cloud.region` / `resource.category` / `resource.instance.type`) —
the only universally-populated attribution axis (CC-0/CC-2).

### AWS

```
/dt-cloud-cost CLOUD:aws TAG:CostCenter
/dt-cloud-cost CLOUD:aws TAG:Team SCOPE:HOST-XXXXXXXX
/dt-cloud-cost CLOUD:aws TAG:Application SCOPE:us-east-1 -clean
/dt-cloud-cost CLOUD:aws TAG:Project --review --pdf
```

### Azure

```
/dt-cloud-cost CLOUD:azure TAG:dt_owner_team
/dt-cloud-cost CLOUD:azure TAG:CostCenter SCOPE:westeurope
/dt-cloud-cost CLOUD:azure TAG:Environment -clean
/dt-cloud-cost CLOUD:azure TAG:project
```

### GCP

```
/dt-cloud-cost CLOUD:gcp TAG:team
/dt-cloud-cost CLOUD:gcp TAG:cost-center SCOPE:us-central1
/dt-cloud-cost CLOUD:gcp TAG:application -clean
/dt-cloud-cost CLOUD:gcp TAG:environment
```

## Arguments

- `CLOUD:<aws|azure|gcp>` — REQUIRED. Filters bizevents on
  `cloud.provider == "{cloud}"` and selects the sub-skill reference set the
  cost workers will read.
- `TAG:<key>` — **OPTIONAL** (CC-0). The OPTIONAL tag-attribution overlay key.
  The carbon-app bizevents do NOT carry tag fields directly; when a TAG is
  supplied the worker resolves each `dt.entity.host` against the Smartscape HOST
  node's `tags[]` to extract the tag value, then groups spend by that value.
  **When TAG is omitted — or when the host join resolves zero rows (which is the
  case on tenants where `dt.entity.host` is NULL on cost events) — the skill
  produces a DIMENSIONAL report** (`cloud.region` / `resource.category` /
  `resource.instance.type`) instead of aborting. TAG never gates the run.
- `SCOPE:<region|HOST-id>` — OPTIONAL. v1 does NOT support native cloud account
  IDs / subscription UUIDs / project IDs as scope anchors — the carbon-app
  bizevents do not carry those fields. SCOPE accepts either:
  - a `cloud.region` value (e.g. `us-east-1`, `westeurope`, `us-central1`) →
    filters `cloud.region == "{SCOPE}"`, OR
  - a Smartscape HOST id (`HOST-XXXXXXXX`) → filters
    `dt.entity.host == "{SCOPE}"`.
  When omitted, cost rolls up across the entire CLOUD provider.
- `--spend` — Optional. Forces the spend source to `cost.list.spend` (realized
  spend via `cost.total`) instead of the default `cost.list.price` (catalog list
  price via `price.total`). **Default is `cost.list.price`** (CC-1) because it
  is the populated feed on the validated tenant; the Phase 0b.0 probe overrides
  this default at runtime to whichever USD-bearing type is actually populated.
- `--list-price` — DEPRECATED no-op alias retained for backward compatibility
  (price is already the default). Accepted silently.
- `--pdf` — Optional. Also render a PDF after the Markdown report. **Default is
  OFF — Markdown only** (Mandate 1). A missing PDF engine logs one line and
  continues; it never fails the run. Legacy `pdf=true` is an alias for `--pdf`;
  `pdf=false` is a no-op.
- `-clean` — Sanitize all identifying names (host IDs, region, tag values,
  resource names, tenant URL). Same rules as `/dt-rca` Phase 1.14.
- `--review` — Add a reviewer subagent quality gate before saving the report.
- `PERIOD:<current|YYYY-MM>` — Optional. Override the current period
  (defaults to current calendar month). Prior period auto-derived as the
  immediately preceding calendar month.

> **Speed note:** run at **medium effort**. The cost-attribution math is bounded
> and deterministic; high effort multiplies latency without quality gains.

---

## Sub-Skills Loaded Per Phase

dt-cloud-cost carries NO inventory DQL of its own. Each worker reads ONE targeted
reference file at start, once per subagent, derives queries from it, and applies
the dt-cloud-cost scoping rules (SCOPE filter + TAG group-by + billing period).
This keeps DQL current — if a field or metric changes upstream, only the
reference needs fixing.

| Phase | Cloud | Reference file (read ONCE at worker start) |
|---|---|---|
| W-cost | aws | `~/.claude/skills/dt-obs-aws/references/resource-ownership.md` + `cost-optimization.md` |
| W-cost | azure | `~/.claude/skills/dt-obs-azure/references/resource-ownership.md` + `cost-optimization.md` |
| W-cost | gcp | `~/.claude/skills/dt-obs-gcp/references/resource-ownership.md` + `resource-management.md` |
| W-baseline | (same as W-cost) | (same files; different time window) |
| W-movers | (same as W-cost) | (joins W-cost and W-baseline results) |
| W-attribution | (same as W-cost) | + entity-specific service queries from `cost-optimization.md` |

**Billing events authority:** Grail `fetch bizevents` filtered to
`event.provider == "dynatrace.biz.carbon"`. This was tenant-validated
2026-05, re-validated 2026-06-03 — there is NO `fetch billing` data object and
NO `billing.cost.usd` field on Grail. USD-bearing event types are
`cost.list.price` (**DEFAULT** as of 2026-06-03, field `price.total`) and
`cost.list.spend` (opt-in via `--spend`, field `cost.total`); the Phase 0b.0
probe auto-detects which is populated. Validated populated dimensions on these
events: `cloud.provider`, `cloud.region`, `resource.category`,
`resource.instance.type`. The `dt.entity.host` field exists in the schema but is
**NULL on every cost event on the validated tenant** — so PRIMARY attribution is
by the native dimensions above. Team / cost-center attribution via joining
`dt.entity.host` against the Smartscape HOST node's `tags[]` is an OPTIONAL
overlay that only works when that field is populated AND hosts carry the tag.

---

## Skill Registry

**Maintenance-time reference map.** Used when updating the skill or as a per-query
fallback if a field/metric name errors at run time. NOT read during normal runs.
Never guess a path, metric, or field name — fix it here and re-validate.

### DQL Authority

| File | Purpose | When to read |
|------|---------|--------------|
| `~/.claude/skills/dtctl/references/DQL-reference.md` | Core DQL syntax, filter patterns, aggregation | Always (Phase 0c) |
| `~/.claude/skills/dt-dql-essentials/references/dql/dql-functions-smartscape.md` | smartscapeNodes, getNodeName, getNodeField signatures | Always (Phase 0c) |

### W-cost / W-baseline / W-movers / W-attribution

| Cloud | File | When to read |
|------|------|--------------|
| aws | `~/.claude/skills/dt-obs-aws/SKILL.md` | Always |
| aws | `~/.claude/skills/dt-obs-aws/references/resource-ownership.md` | Always (TAG group-by patterns) |
| aws | `~/.claude/skills/dt-obs-aws/references/cost-optimization.md` | Always (resource inventory; idle detection) |
| azure | `~/.claude/skills/dt-obs-azure/SKILL.md` | Always |
| azure | `~/.claude/skills/dt-obs-azure/references/resource-ownership.md` | Always |
| azure | `~/.claude/skills/dt-obs-azure/references/cost-optimization.md` | Always |
| gcp | `~/.claude/skills/dt-obs-gcp/SKILL.md` | Always |
| gcp | `~/.claude/skills/dt-obs-gcp/references/resource-ownership.md` | Always (gcp_labels pattern) |
| gcp | `~/.claude/skills/dt-obs-gcp/references/resource-management.md` | Always (project + region rollups) |

**Sanitization, URL format, Mermaid/ASCII diagram rules, and PDF generation:**
These are identical to `/dt-rca`. Read `~/.claude/skills/dt-rca/SKILL.md` sections
"Phase 1.14", "Phase 3", "Phase 4", "PDF DIAGRAM RULES", "MERMAID SYNTAX RULES",
and "DIAGRAM SIZING RULES" — all apply unchanged.

---

## ENTITY MODEL — CARBON-APP BIZEVENTS + SMARTSCAPE HOST JOIN

dt-cloud-cost reads two substrates and joins them. **Tenant-validated 2026-05.**

1. **Cost numbers** come from `fetch bizevents | filter event.provider ==
   "dynatrace.biz.carbon"`. NO `classicEntitySelector()`. NO `fetch billing`.
2. **Tag values** come from `smartscapeNodes "HOST"` — the bizevents carry
   `dt.entity.host` but NOT `tags[]`. Tag attribution = join.

### Validated Bizevent Schema

| Field | Populated on event.type | Notes |
|-------|-------------------------|-------|
| `event.provider` | all | always `"dynatrace.biz.carbon"` |
| `event.type` | all | `cost.list.spend` / `cost.list.price` / `carbon.measurement.*` |
| `cost.total` | `cost.list.spend` | USD spend (numeric) — DEFAULT cost field |
| `cost.currency` | `cost.list.spend` | `"USD"` |
| `price.total` | `cost.list.price` | USD list price (numeric) — `--list-price` mode |
| `price.currency` | `cost.list.price` | `"USD"` |
| `cloud.provider` | all | `aws` / `azure` / `gcp` |
| `cloud.region` | all | e.g. `us-east-1`, `westeurope`, `us-central1` |
| `resource.category` | all | e.g. `instance`, `storage`, `network` |
| `resource.instance.type` | all | e.g. `e96adsv6`, `m5.large` |
| `dt.entity.host` | all | `HOST-...` — join key to Smartscape |

Carbon-emission events (`event.type` starts with `carbon.measurement`) carry
emissions data, not USD. The skill must filter them OUT.

### Cost Source Selection (sourced from DispatchContext)

The Phase 0b.0 Substrate Probe sets these at runtime by counting populated rows
per event.type. The values below are the static fallbacks if the probe cannot
run; the probe ALWAYS wins when it returns a populated type.

```
default (no flag)        → COST_EVENT_TYPE = "cost.list.price"   (CC-1)
                            COST_FIELD      = "price.total"
                            COST_CURRENCY   = "price.currency"
--spend                  → COST_EVENT_TYPE = "cost.list.spend"
                            COST_FIELD      = "cost.total"
                            COST_CURRENCY   = "cost.currency"
--list-price             → no-op alias (price is already the default)

PROBE OVERRIDE: Phase 0b.0 counts both types; if exactly one is populated it
binds that one (and its field/currency). If both are populated, the flag (or the
price default) decides. If neither is populated, the Billing-Empty Gate fires.
```

### Scope Filter

```dql
-- Optional region scope
| filter cloud.region == "{SCOPE}"

-- Optional host scope
| filter dt.entity.host == "{SCOPE}"

-- No SCOPE → no host/region filter; CLOUD filter still applies
```

### Attribution Mode (CC-2 — decided in Phase 0b.0)

```
ATTRIBUTION_MODE = "dimensional"   (PRIMARY — always works)
                 | "tag"           (OPTIONAL overlay — only when ALL hold:
                                     1. TAG_KEY was supplied, AND
                                     2. dt.entity.host is populated on cost events
                                        (host_populated probe > 0), AND
                                     3. the host→tag join resolves ≥1 row)

Decision: Phase 0b.0 probes `countIf(isNotNull(dt.entity.host))` on cost events.
  host_populated == 0  → ATTRIBUTION_MODE = "dimensional" (tag overlay impossible)
  host_populated  > 0  and TAG_KEY present → attempt "tag"; if the join yields 0
                         rows, fall back to "dimensional" and note it.
  TAG_KEY absent       → ATTRIBUTION_MODE = "dimensional".
```

The dimensional rollup by `cloud.region` / `resource.category` /
`resource.instance.type` is the canonical attribution table when
ATTRIBUTION_MODE == "dimensional". The tag column simply does not appear.

### Tag Value Resolution (Smartscape HOST join — OPTIONAL overlay)

Used ONLY when ATTRIBUTION_MODE == "tag". Bizevents do not carry `tags[]`. Build
a host→tag-value lookup from Smartscape, then `lookup` it into the bizevent query.

```dql
-- VALIDATED PATTERN — host tag lookup
smartscapeNodes "HOST"
| filter isNotNull(tags[`{TAG_KEY}`])
| fields id, tag_value = tags[`{TAG_KEY}`]
```

This lookup is joined to the cost query in tag mode (see W-cost queries below).

### Cost Queries — Validated Substrate

```dql
-- PRIMARY (ATTRIBUTION_MODE == "dimensional"): native event dimensions, no join.
-- VALIDATED 2026-06-03: returns the USD 152K Azure spend; works when dt.entity.host
-- is NULL on cost events (the validated tenant's reality).
fetch bizevents, from: toTimestamp("{PERIOD_START}"), to: toTimestamp("{PERIOD_END}")
| filter event.provider == "dynatrace.biz.carbon"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
// optional scope:
// | filter cloud.region == "{SCOPE}"   OR   | filter dt.entity.host == "{SCOPE}"
| summarize cost_usd = sum({COST_FIELD}),
            by: { cloud.region, resource.category, resource.instance.type }
| sort cost_usd desc
| limit 20
```

```dql
-- OPTIONAL OVERLAY (ATTRIBUTION_MODE == "tag"): host→tag join. Only run when the
-- Phase 0b.0 host-populated probe > 0 AND a TAG was supplied. On the validated
-- tenant this yields zero rows (dt.entity.host is NULL), which is WHY it is not
-- the primary path. Falls back to dimensional if the join is empty.
fetch bizevents, from: toTimestamp("{PERIOD_START}"), to: toTimestamp("{PERIOD_END}")
| filter event.provider == "dynatrace.biz.carbon"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
// optional scope:
// | filter cloud.region == "{SCOPE}"   OR   | filter dt.entity.host == "{SCOPE}"
| lookup [
    smartscapeNodes "HOST"
    | filter isNotNull(tags[`{TAG_KEY}`])
    | fields id, tag_value = tags[`{TAG_KEY}`]
  ], sourceField: dt.entity.host, lookupField: id, prefix: "host_"
| filter isNotNull(host_tag_value)
| summarize cost_usd = sum({COST_FIELD}),
            by: { tag_value = host_tag_value }
| sort cost_usd desc
| limit 20
```

> **Schema absence probe.** If the cost query returns FIELD_DOES_NOT_EXIST or
> UNKNOWN_DATA_OBJECT, the worker re-runs ONCE:
> `fetch bizevents, from:now()-7d | filter event.provider ==
> "dynatrace.biz.carbon" | summarize count(), by: { event.type }` — this
> confirms whether the carbon-app is ingesting at all, and which event types
> are present. The worker records the discovered schema in `field_discovery`
> and stops rather than guessing alternative field paths.

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking — set up at the very start)

Maintain a per-run timing log. Set `LOG="COST_{CLOUD}_{SCOPE_SLUG}_{DATE}.log"`
(next to the report) and at each phase boundary append ONE line, backgrounded so
it never gates execution:
```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line summary>" >> "$LOG"; } &
```
Log: START, Phase 0b complete (DispatchContext summary), Phase 1 complete
(per-worker one-liners with row counts), Billing-Empty-Gate result, report
written, RUN COMPLETE. Phase 7 prints the log path.

---

### Phase 0-auth: PRE-WARM the dtctl session (Orchestrator — proactive step 0)

**FIRST action of the whole run — before any data query and before any dispatch:**
```
dtctl auth refresh
```
This is a REFRESH, not a login — silent, no browser, uses the on-disk refresh
token (`DTCTL_TOKEN_STORAGE=file`, harness env, never the Keychain). Pre-warming
guarantees a fresh access token up front and prevents the parallel workers from
each triggering a concurrent refresh race.

- `dtctl auth refresh` succeeds → token fresh for the entire run; proceed to Phase 0a.
- It fails (no/expired refresh token — first-ever use, or weeks idle) → ONLY THEN run once:
  `dtctl auth login --plain --safety-level readonly`.

**Workers never authenticate.** They inherit the pre-warmed on-disk token. If a
worker still returns `<gap: auth>`, the orchestrator refreshes once and
re-dispatches only that worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

---

### Phase 0a: Argument Parser

```
CLOUD       = value of CLOUD:<aws|azure|gcp>  → REQUIRED
TAG         = value of TAG:<key>               → OPTIONAL (CC-0). If absent,
              ATTRIBUTION_MODE starts as "dimensional".
SCOPE       = value of SCOPE:<region|HOST-id>  → OPTIONAL (region or HOST-...)
PERIOD      = value of PERIOD:<current|YYYY-MM> (default: current calendar month)

FLAGS:
  -clean         → CLEAN_MODE = true
  --review       → REVIEW_MODE = true
  --pdf          → PDF_MODE = true                (DEFAULT FALSE — Mandate 1)
                   (legacy `pdf=true` is an alias for --pdf; `pdf=false` is a no-op)
  --spend        → SPEND_MODE = true              (force cost.list.spend)
  --list-price   → no-op alias (price is already the default; accept silently)

VALIDATION:
  - CLOUD missing or not in {aws, azure, gcp} → print usage and exit
  - TAG is OPTIONAL (CC-0) — a missing TAG is NOT an error; the run proceeds in
    dimensional attribution mode. NEVER print usage / exit for a missing TAG.
  - SCOPE optional; if present, classify as HOST_ID (starts with "HOST-")
    or REGION (everything else).
```

---

### Phase 0b: Substrate Validation + Period Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** that all phase workers receive.

#### Step 0b.0 — Substrate Probe (Mandate 2 — runs ONCE, BEFORE everything else)

The carbon-app event schema drifts across tenants. Probe the real shape at
runtime and bind the working values into the DispatchContext, rather than
trusting the static defaults. ONE cheap query does it all — **note that
`dt.entity.host` population is measured PER event.type, because the two feeds can
differ** (validated: it is NULL on `cost.list.price` but present on the tiny
`cost.list.spend` feed):

```dql
fetch bizevents, from: now() - 7d
| filter event.provider == "dynatrace.biz.carbon"
| filter cloud.provider == "{CLOUD}"
| summarize
    price_events       = countIf(event.type == "cost.list.price"),
    spend_events       = countIf(event.type == "cost.list.spend"),
    price_host_populated = countIf(event.type == "cost.list.price" and isNotNull(dt.entity.host)),
    spend_host_populated = countIf(event.type == "cost.list.spend" and isNotNull(dt.entity.host))
```

**(a) Cost event-type / field selection — CANDIDATE LIST `[cost.list.price (PRIMARY, 2026-06-03), cost.list.spend (legacy)]`:**
```
if SPEND_MODE and spend_events > 0      → COST_EVENT_TYPE = "cost.list.spend",
                                           COST_FIELD = "cost.total",  COST_CURRENCY_FIELD = "cost.currency"
elif price_events > 0                    → COST_EVENT_TYPE = "cost.list.price",  (DEFAULT, CC-1)
                                           COST_FIELD = "price.total", COST_CURRENCY_FIELD = "price.currency"
elif spend_events > 0                    → COST_EVENT_TYPE = "cost.list.spend",
                                           COST_FIELD = "cost.total",  COST_CURRENCY_FIELD = "cost.currency"
else (both zero)                         → no USD-bearing type populated; the
                                           Phase 1.5 Billing-Empty Gate will fire.
```
Pick the FIRST populated candidate in priority order (price first, unless
`--spend` forced spend and spend is populated). This is the self-heal: if the
tenant flips which feed it populates, the skill follows.

**(b) Attribution mode — host-population gate, measured ON THE SELECTED FEED (CC-2):**
```
HOST_POPULATED = (the *_host_populated counter for the SELECTED COST_EVENT_TYPE > 0)
  e.g. COST_EVENT_TYPE=="cost.list.price" → HOST_POPULATED = (price_host_populated > 0)
       COST_EVENT_TYPE=="cost.list.spend" → HOST_POPULATED = (spend_host_populated > 0)

if not HOST_POPULATED                    → ATTRIBUTION_MODE = "dimensional"
                                           (tag overlay is impossible on this feed;
                                           do NOT abort)
elif TAG present and HOST_POPULATED      → ATTRIBUTION_MODE = "tag"  (overlay will
                                           be attempted; W-cost falls back to
                                           dimensional if the join yields 0 rows)
else (TAG absent)                        → ATTRIBUTION_MODE = "dimensional"
```
> CRITICAL: do NOT gate on host-population across ALL feeds. The cheap
> `cost.list.spend` feed may carry `dt.entity.host` while the populated
> `cost.list.price` feed does not — gating on the wrong feed would wrongly enable
> tag mode against a feed that has no host key.

**Emit a one-line probe summary to the run log**, e.g.:
`Probe: event.type→cost.list.price (price_events=79215, spend_events=843), price_host=0 → ATTRIBUTION_MODE=dimensional`

> On the validated `demo` tenant (Azure, 2026-06-03): the price feed is the
> populated one (`price_events`≈33963 over 3d / ≈79215 over 7d; `spend_events`≈361
> over 3d / ≈843 over 7d) and `price_host_populated == 0` → binds
> `cost.list.price`/`price.total` and `ATTRIBUTION_MODE=dimensional`. (The spend
> feed DOES carry `dt.entity.host`, but it is near-empty — USD 68 — so it is not
> selected, and its host key is irrelevant.) This is the expected, correct outcome.

#### Step 1 — Classify SCOPE (if provided)

```
SCOPE absent → SCOPE_KIND = "none",   SCOPE_FILTER = ""
SCOPE matches ^HOST-[A-F0-9]{16}$ → SCOPE_KIND = "host",
                                     SCOPE_FILTER = '| filter dt.entity.host == "{SCOPE}"'
Otherwise → SCOPE_KIND = "region",
            SCOPE_FILTER = '| filter cloud.region == "{SCOPE}"'

(No native cloud account ID / subscription UUID / project ID validation —
those fields are NOT carried by the carbon-app bizevents in v1.)
```

#### Step 2 — Confirm carbon-app ingestion is live for the CLOUD

```dql
fetch bizevents, from: now() - 7d
| filter event.provider == "dynatrace.biz.carbon"
| filter cloud.provider == "{CLOUD}"
| summarize event_count = count(), distinct_hosts = countDistinct(dt.entity.host)
```

If `event_count == 0`: print a clear "carbon-app not ingesting for CLOUD={CLOUD}"
message and exit. The dt-obs-{CLOUD}/SKILL.md carbon-app onboarding checklist
is the canonical remediation path.

If SCOPE_KIND != "none", re-run with the SCOPE_FILTER appended and require
`event_count > 0` for the scoped query as well.

#### Step 3 — Derive current + prior period bounds

```
PERIOD == "current" (default):
  CURRENT_START = first day of current calendar month at 00:00 UTC
  CURRENT_END   = now()
PERIOD == "YYYY-MM":
  CURRENT_START = first day of that month at 00:00 UTC
  CURRENT_END   = first day of NEXT month at 00:00 UTC (or now() if same as current)

PRIOR_START = first day of immediately preceding calendar month at 00:00 UTC
PRIOR_END   = first day of the CURRENT month at 00:00 UTC

Store all four as ISO-8601 strings.
```

Store everything as **DispatchContext JSON**:

```json
{
  "CLOUD": "aws|azure|gcp",
  "SCOPE": "us-east-1 | HOST-XXXXXXXX | null",
  "SCOPE_KIND": "region | host | none",
  "SCOPE_FILTER": "| filter cloud.region == \"...\"  |  | filter dt.entity.host == \"HOST-...\"  |  \"\"",
  "TAG_KEY": "... | null",
  "ATTRIBUTION_MODE": "dimensional | tag",
  "HOST_POPULATED": false,
  "TAG_LOOKUP_DQL": "smartscapeNodes \"HOST\" | filter isNotNull(tags[`{TAG_KEY}`]) | fields id, tag_value = tags[`{TAG_KEY}`]",
  "EVENT_PROVIDER": "dynatrace.biz.carbon",
  "COST_EVENT_TYPE": "cost.list.price | cost.list.spend",
  "COST_FIELD": "price.total | cost.total",
  "COST_CURRENCY_FIELD": "price.currency | cost.currency",
  "SPEND_MODE": false,
  "PDF_MODE": false,
  "PROBE_SUMMARY": "event.type→cost.list.price (price_events=N, spend_events=M), host_populated=K → ATTRIBUTION_MODE=...",
  "CURRENT_PERIOD": { "start": "ISO", "end": "ISO", "label": "YYYY-MM" },
  "PRIOR_PERIOD":   { "start": "ISO", "end": "ISO", "label": "YYYY-MM" },
  "CLEAN_MODE": false,
  "REVIEW_MODE": false
}
```

> `COST_EVENT_TYPE` / `COST_FIELD` / `COST_CURRENCY_FIELD` / `ATTRIBUTION_MODE` /
> `HOST_POPULATED` are all bound by the **Phase 0b.0 Substrate Probe** above —
> not hardcoded. `TAG_KEY` may be `null` (dimensional mode).

---

### Phase 0c: Reference Strategy

dt-cloud-cost is **reference-driven** — workers carry no hardcoded cost DQL.
Each worker reads ONE targeted reference file (its `cost-optimization.md` /
`resource-ownership.md` for the chosen CLOUD), once per subagent, derives the
query patterns from it, and applies the dt-cloud-cost scoping (SCOPE filter,
TAG group-by, billing period window). This is the correctness design: DQL stays
current automatically, drift is impossible.

**Orchestrator exceptions (inline only):** Phase 0b scope-existence probe and
Phase 1.5 Billing-Empty-Gate remain inline — they are orchestration primitives,
tiny, stable, and must run BEFORE workers dispatch.

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

**Dispatch ALL workers in ONE message with multiple Agent calls — true
concurrency.** Dispatching them one-at-a-time defeats the architecture and is a
top cause of slow runs. The whole point is that all four workers run AT THE SAME
TIME.

Workers run concurrently as subagents (`subagent_type: Explore`).

**Worker-failure handling — NO silent serial fallback.** If a worker returns
gaps or fails, do NOT re-run all of its queries yourself inline in the
orchestrator thread. Instead: **re-dispatch that ONE worker ONCE** (as a fresh
Agent call, with a note on what to fix), and if it still fails, record a
`⚠️ gap` for that section in the report.

#### Worker Prompt Template

```
You are a cost-attribution worker for dt-cloud-cost (Dynatrace FinOps Reporter).

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read the reference file(s) listed in "REFERENCE FILES" below. Read each file ONCE.
Do not read any other files. Do not re-read. Take the query patterns from the
reference; apply the DISPATCH CONTEXT scoping (SCOPE filter, TAG group-by,
billing period window). This is the ONLY source of DQL truth — do NOT derive
queries from training knowledge or guess field names.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple `dtctl query` tool calls,
in parallel). Do not read files and query at the same time — read first, then
query.

ENTITY MODEL — CRITICAL:
- NEVER use classicEntitySelector(). NEVER use `fetch billing`. NEVER use
  `billing.cost.usd`. These do NOT exist in v1's tenant-validated substrate.
- Cost queries: `fetch bizevents | filter event.provider == "{EVENT_PROVIDER}"
  | filter event.type == "{COST_EVENT_TYPE}" | filter cloud.provider ==
  "{CLOUD}" {SCOPE_FILTER}` — sum `{COST_FIELD}` exactly as given.
- ATTRIBUTION (read {ATTRIBUTION_MODE} from the DISPATCH CONTEXT):
  - "dimensional" (PRIMARY / default): group by native event fields
    `{ cloud.region, resource.category, resource.instance.type }`. NO tag join.
    This is the only attribution axis that works when `dt.entity.host` is NULL
    on cost events (the validated tenant's reality). NEVER attempt the host→tag
    join in dimensional mode.
  - "tag" (OPTIONAL overlay): ONLY when {ATTRIBUTION_MODE}=="tag". bizevents carry
    NO tags — JOIN via `lookup [{TAG_LOOKUP_DQL}], sourceField: dt.entity.host,
    lookupField: id, prefix: "host_"`, then group by `host_tag_value`. If this
    join returns ZERO rows (a SUCCESSFUL query with no rows = real absence), set
    `tag_join_empty=true` in your result and ALSO return the dimensional rollup
    as the usable attribution table. Do NOT fail — degrade to dimensional.
- Carbon-emission events (event.type starts with "carbon.measurement") must
  be EXCLUDED — they carry emissions, not USD.

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

QUERIES TO RUN:
{QUERY_INSTRUCTIONS_FOR_THIS_WORKER}

RULES:
- EXPLORATION CAP: run ONLY the queries defined in "QUERIES TO RUN". No ad-hoc
  "let me also check…" queries. If a specific finding requires one additional
  query, use a pattern from your already-loaded reference — never a free guess.
- PREFLIGHT: run `DTCTL_TOKEN_STORAGE=file dtctl query 'fetch bizevents | limit 1'`.
  If it errors with an auth/connection problem (not a DQL error), return
  immediately with `<gap: dtctl-unavailable: {message}>` and STOP.
- BILLING-EVENT SCHEMA DISCOVERY: if the cost query returns FIELD_DOES_NOT_EXIST
  or UNKNOWN_DATA_OBJECT, re-run ONCE with
  `fetch bizevents, from:now()-7d | filter event.provider == "dynatrace.biz.carbon"
   | summarize count(), by: { event.type } | limit 20`
  to confirm the carbon-app is ingesting and which event types are present.
  Record findings in `field_discovery`; do NOT guess alternative field paths
  (no `billing.cost.usd`, no `fetch billing`, no `{cloud}.billing` event types).
- Run all independent queries in parallel (multiple tool calls per message).
- Do NOT return raw rows. Internalize; summarize aggressively.
- Cap your PhaseResult at ~4KB. Top-20 of anything, not top-200.
- If a query fails with a SYNTAX/FIELD error, retry ONCE with corrected syntax
  (check: aliased aggregations not backticked; single-quoted dtctl arg).
- AUTH IS THE ORCHESTRATOR'S JOB — workers NEVER run `dtctl auth login` OR
  `dtctl auth refresh`. If a dtctl call returns an auth error, verify the
  `DTCTL_TOKEN_STORAGE=file` prefix is present on EVERY dtctl call. If the
  prefix is present and it still fails, return `<gap: auth>` and STOP — never
  attempt any auth recovery.
- ERROR ≠ ABSENCE: a query that ERRORED tells you nothing about whether data
  exists. Only a SUCCESSFUL query returning zero rows is evidence of absence.
- Execute DQL via `DTCTL_TOKEN_STORAGE=file dtctl query '<dql>'` — prefix EVERY
  dtctl call with `DTCTL_TOKEN_STORAGE=file`. SINGLE QUOTES around the DQL.
- NO BACKTICKS for sort aliases. Alias aggregations (`cost_usd = sum(...)
  | sort cost_usd desc`) — never `sort \`sum(...)\``.

RETURN only this PhaseResult shape (JSON). Nothing else.
{PHASE_RESULT_SCHEMA}
```

---

#### W-cost — Current Period Spend (dimensional PRIMARY; tag overlay optional)

**REFERENCE-DRIVEN — read the `resource-ownership.md` + `cost-optimization.md`
for your CLOUD ONCE** for the Smartscape HOST tag patterns and resource-type
context. Cost numbers come from carbon-app bizevents per the "ENTITY MODEL"
block above — tenant-validated.

**dt-cloud-cost scoping/hygiene to apply to EVERY query:**
- Outer filters: `event.provider == "{EVENT_PROVIDER}"`,
  `event.type == "{COST_EVENT_TYPE}"`, `cloud.provider == "{CLOUD}"`.
- Optional scope: append `{SCOPE_FILTER}` (empty string if SCOPE_KIND=="none").
- Window: `from: toTimestamp("{CURRENT_PERIOD.start}"), to: toTimestamp("{CURRENT_PERIOD.end}")`.
- Alias every aggregation (`cost_usd = sum({COST_FIELD})`), single-quote the dtctl arg.

**Queries to run (ALL in one parallel batch):**

1. **Dimensional rollup — ALWAYS RUN (the PRIMARY attribution table)** — group by
   native event fields; no join. Top 20 by `cost_usd`:
   ```
   fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
   | filter event.provider == "{EVENT_PROVIDER}"
   | filter event.type == "{COST_EVENT_TYPE}"
   | filter cloud.provider == "{CLOUD}"
   {SCOPE_FILTER}
   | summarize cost_usd = sum({COST_FIELD}),
               by: { cloud.region, resource.category, resource.instance.type }
   | sort cost_usd desc | limit 20
   ```
2. **Total spend (sanity)** — drop the group-by; one row.
   `summarize cost_usd = sum({COST_FIELD})`. (Validated Azure 3d: ~USD 152,091.)
3. **Spend by TAG value (top 20) — RUN ONLY IF {ATTRIBUTION_MODE}=="tag"** —
   host-tag join pattern. If the join returns 0 rows, set `tag_join_empty=true`
   and rely on Q1 for attribution (do NOT fail):
   ```
   fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
   | filter event.provider == "{EVENT_PROVIDER}"
   | filter event.type == "{COST_EVENT_TYPE}"
   | filter cloud.provider == "{CLOUD}"
   {SCOPE_FILTER}
   | lookup [{TAG_LOOKUP_DQL}], sourceField: dt.entity.host, lookupField: id, prefix: "host_"
   | filter isNotNull(host_tag_value)
   | summarize cost_usd = sum({COST_FIELD}), by: { tag_value = host_tag_value }
   | sort cost_usd desc | limit 20
   ```
4. **Untagged spend (gap-check) — RUN ONLY IF {ATTRIBUTION_MODE}=="tag"** — same
   as Q3 but `filter isNull(host_tag_value)`; one row. Chargeback compliance gap.

**PhaseResult shape:**
```json
{
  "period": { "start": "ISO", "end": "ISO", "label": "YYYY-MM" },
  "attribution_mode": "dimensional | tag",
  "total_cost_usd": 0,
  "dimensional_rollup": [
    { "cloud_region": "...", "resource_category": "...", "resource_instance_type": "...", "cost_usd": 0, "pct_of_total": 0 }
  ],
  "tag_join_empty": false,
  "tagged_cost_usd": 0,
  "untagged_cost_usd": 0,
  "untagged_pct": 0,
  "top_owners": [
    { "tag_value": "...", "cost_usd": 0, "pct_of_total": 0 }
  ],
  "field_discovery": { "billing_event_type_actual": "...", "cost_field_actual": "..." },
  "gaps": []
}
```

> In dimensional mode `top_owners` / `tagged_cost_usd` / `untagged_*` are empty/0
> and `dimensional_rollup` carries the attribution. In tag mode that fell back
> (`tag_join_empty==true`), treat it as dimensional for the report.

---

#### W-baseline — Prior Period Spend (dimensional PRIMARY; tag overlay optional)

**REFERENCE-DRIVEN — same files as W-cost.** Identical query shapes; different
time window only.

**dt-cloud-cost scoping/hygiene:**
- Window: `from: toTimestamp("{PRIOR_PERIOD.start}"), to: toTimestamp("{PRIOR_PERIOD.end}")`
- All other filters identical to W-cost.

**Queries to run (ALL in one parallel batch):**

1. **Prior-period dimensional rollup (top 20) — ALWAYS RUN** — same shape as
   W-cost Q1, prior window.
2. **Prior-period total spend — ALWAYS RUN** — same shape as W-cost Q2, prior window.
3. **Prior-period spend by TAG value — RUN ONLY IF {ATTRIBUTION_MODE}=="tag"** —
   same shape as W-cost Q3, prior window.
4. **Prior-period untagged spend — RUN ONLY IF {ATTRIBUTION_MODE}=="tag"** — same
   shape as W-cost Q4, prior window.

**PhaseResult shape:**
```json
{
  "period": { "start": "ISO", "end": "ISO", "label": "YYYY-MM" },
  "attribution_mode": "dimensional | tag",
  "total_cost_usd": 0,
  "dimensional_rollup": [
    { "cloud_region": "...", "resource_category": "...", "resource_instance_type": "...", "cost_usd": 0 }
  ],
  "tag_join_empty": false,
  "tagged_cost_usd": 0,
  "untagged_cost_usd": 0,
  "untagged_pct": 0,
  "top_owners": [
    { "tag_value": "...", "cost_usd": 0, "pct_of_total": 0 }
  ],
  "field_discovery": { },
  "gaps": []
}
```

---

#### W-movers — Top Deltas (Positive AND Negative)

**REFERENCE-DRIVEN — same files; this worker JOINS W-cost and W-baseline.**

W-movers does NOT re-query billing events from scratch. It runs ONE joined query
that computes per-key deltas in a single pass. **The grouping key follows
{ATTRIBUTION_MODE}**: in dimensional mode the key is
`resource.instance.type` (the most actionable native dimension — swap to
`cloud.region` or `resource.category` if a worker reference suggests it); in tag
mode the key is the host-tag value.

**Query — DIMENSIONAL mode (single call, {ATTRIBUTION_MODE}=="dimensional"):**
```dql
fetch bizevents, from:toTimestamp("{PRIOR_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
| filter event.provider == "{EVENT_PROVIDER}"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
{SCOPE_FILTER}
| fieldsAdd period = if(timestamp < toTimestamp("{CURRENT_PERIOD.start}"), "prior", else: "current")
| summarize cost_usd = sum({COST_FIELD}), by: { mover_key = resource.instance.type, period }
```

**Query — TAG mode (single call, {ATTRIBUTION_MODE}=="tag"):**
```dql
fetch bizevents, from:toTimestamp("{PRIOR_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
| filter event.provider == "{EVENT_PROVIDER}"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
{SCOPE_FILTER}
| lookup [{TAG_LOOKUP_DQL}], sourceField: dt.entity.host, lookupField: id, prefix: "host_"
| filter isNotNull(host_tag_value)
| fieldsAdd period = if(timestamp < toTimestamp("{CURRENT_PERIOD.start}"), "prior", else: "current")
| summarize cost_usd = sum({COST_FIELD}), by: { mover_key = host_tag_value, period }
```

The worker pivots `period` into `prior_usd` / `current_usd` in code, computes
`delta_usd = current_usd - prior_usd` and `delta_pct = delta_usd / prior_usd`,
then returns the top 10 by absolute `delta_usd` (positive and negative both
included; sort by abs descending). `mover_key` is the dimension value (dimensional
mode) or the tag value (tag mode).

**PhaseResult shape:**
```json
{
  "mover_key_kind": "resource.instance.type | tag",
  "top_positive_movers": [
    { "mover_key": "...", "prior_usd": 0, "current_usd": 0, "delta_usd": 0, "delta_pct": 0 }
  ],
  "top_negative_movers": [
    { "mover_key": "...", "prior_usd": 0, "current_usd": 0, "delta_usd": 0, "delta_pct": 0 }
  ],
  "new_keys":     [{ "mover_key": "...", "current_usd": 0 }],
  "dropped_keys": [{ "mover_key": "...", "prior_usd":   0 }],
  "gaps": []
}
```

`new_keys` = keys present in current but missing from prior.
`dropped_keys` = keys present in prior but missing from current.
Cap each list at 10 entries.

---

#### W-attribution — Per-Owner Resource Detail

**REFERENCE-DRIVEN — read the same `cost-optimization.md` (entity-type catalog)
+ `resource-ownership.md` (service-specific ownership patterns) for your CLOUD.**

**Mode-aware:**
- **DIMENSIONAL mode ({ATTRIBUTION_MODE}=="dimensional"):** there are no tag
  owners. For each of the top 5 `cloud.region` values from W-cost's dimensional
  rollup, return the per-region resource-category × instance-type breakdown
  (native fields, no join). This populates the "Per-Owner Resource Detail"
  section with REGION as the owner axis. Single query, group by region first:
  ```
  fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
  | filter event.provider == "{EVENT_PROVIDER}"
  | filter event.type == "{COST_EVENT_TYPE}"
  | filter cloud.provider == "{CLOUD}"
  {SCOPE_FILTER}
  | summarize cost_usd = sum({COST_FIELD}),
              by: { cloud.region, resource.category, resource.instance.type }
  | sort cost_usd desc | limit 50
  ```
  Bucket the rows by `cloud.region` in code; emit per-region breakdowns.
- **TAG mode ({ATTRIBUTION_MODE}=="tag"):** the original per-tag-value detail
  below. For each of the top 5 tag values from W-cost (`top_owners[0..4]`),
  discover the contributing hosts + resource categories via the carbon-app
  bizevents joined to Smartscape.

**dt-cloud-cost scoping/hygiene (TAG mode, per tag value):**
- Resolve hosts carrying the tag value: `smartscapeNodes "HOST" | filter
  tags[`{TAG_KEY}`] == "{tag_value}" | fields id, name = entity.name`.
- Spend-per-host query (bizevents):
  ```
  fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
  | filter event.provider == "{EVENT_PROVIDER}"
  | filter event.type == "{COST_EVENT_TYPE}"
  | filter cloud.provider == "{CLOUD}"
  {SCOPE_FILTER}
  | lookup [smartscapeNodes "HOST"
            | filter tags[`{TAG_KEY}`] == "{tag_value}"
            | fields id], sourceField: dt.entity.host, lookupField: id, prefix: "h_"
  | filter isNotNull(h_id)
  | summarize cost_usd = sum({COST_FIELD}),
              by: { resource.category, resource.instance.type }
  | sort cost_usd desc | limit 10
  ```
- Top-host query (same filters, group by `dt.entity.host`, limit 5).

**Queries to run (ALL in one parallel batch — two queries per top tag value,
so up to 10 queries for the 5 owners):**

1. **Resource-category + instance-type breakdown (per top tag value)** — uses
   native bizevent dimensions (no Smartscape join needed beyond the host-tag
   filter). Returns top resource categories contributing to that owner's spend.
2. **Top 5 hosts by spend (per top tag value)** — bizevents grouped by
   `dt.entity.host`. Returns the highest-cost hosts attributed to this owner.

**PhaseResult shape:**
```json
{
  "owner_axis": "cloud.region | tag",
  "per_owner": [
    {
      "owner": "westeurope | <tag_value>",
      "total_usd": 0,
      "resource_category_breakdown": [
        { "resource_category": "instance", "resource_instance_type": "m896ixds32v3", "cost_usd": 0 }
      ],
      "top_hosts": [
        { "host_id": "HOST-...", "host_name": "...", "cost_usd": 0 }
      ]
    }
  ],
  "gaps": []
}
```

> In dimensional mode `owner` is a region and `top_hosts` is empty (host is NULL
> on cost events). In tag mode `owner` is the tag value and `top_hosts` is populated.

---

### Phase 1.5: Billing-Empty Gate (Orchestrator — HARD FAIL if empty)

**Before trusting any worker result, check that carbon-app bizevents exist for
the CLOUD (+ optional SCOPE) in the CURRENT_PERIOD.** This is the binding
constraint baked into dt-cloud-cost: the Dynatrace carbon-impact app must be
ingesting for this cloud, or the report is meaningless.

```dql
fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
| filter event.provider == "{EVENT_PROVIDER}"
| filter event.type == "{COST_EVENT_TYPE}"
| filter cloud.provider == "{CLOUD}"
{SCOPE_FILTER}
| summarize event_count = count(),
            distinct_hosts = countDistinct(dt.entity.host),
            total_usd = sum({COST_FIELD})
```

If `event_count == 0`:
- **FAIL FAST.** Do NOT proceed to Phase 2.
- Print this exact message and exit:
  ```
  Cost report aborted — no carbon-app bizevents found in Grail for
    CLOUD={CLOUD}  SCOPE={SCOPE or "(none)"}  PERIOD={CURRENT_PERIOD.label}
    event.provider="dynatrace.biz.carbon"  event.type="{COST_EVENT_TYPE}".

  The Dynatrace carbon-impact app must be ingesting cost data for {CLOUD}.
  Verify that:
    1. The carbon-impact app is installed and configured in this tenant.
    2. The {CLOUD} provider integration is active inside the app.
    3. Ingestion has caught up to the current period (last_seen < 24h).
    4. If --list-price was passed, the catalog feed is enabled (event.type
       "cost.list.price"); otherwise the spend feed ("cost.list.spend") is
       required.

  See ~/.claude/skills/dt-obs-{CLOUD}/SKILL.md for the carbon-app onboarding
  checklist, or contact your Dynatrace platform owner.
  ```
- Log the failure to the run log and exit with non-zero status.

This gate is non-negotiable — silently producing an empty "0 USD" report would
be worse than failing.

---

### Phase 2: Attribution Synthesis (Orchestrator, Sequential)

After W-cost, W-baseline, W-movers, and W-attribution return (and the
Billing-Empty Gate has cleared), synthesize the attribution narrative.
**The "owner" axis is {ATTRIBUTION_MODE}-dependent:** in dimensional mode the
owner is a (region / resource-category / instance-type) cell or a region; in tag
mode it is a tag value.

The narrative answers four questions, in order:

1. **Who owns the spend?** Top 5 owners from W-cost (dimensional cells or tag
   values) + their share of total.
2. **What changed?** Top movers from W-movers — both growth and savings.
3. **What's new?** `new_keys` from W-movers — new initiatives / first-seen
   instance types or tag values this period.
4. **What dropped off?** `dropped_keys` from W-movers — decommissioned
   workloads, retired instance types, sunset applications.

For each top owner (top 5), pull the W-attribution resource detail to explain
WHAT the owner is paying for (compute vs storage vs network vs database).

Compute these summary metrics for the Executive Summary:
- `total_current_usd` = W-cost.total_cost_usd
- `total_prior_usd`   = W-baseline.total_cost_usd
- `total_delta_usd`   = total_current_usd − total_prior_usd
- `total_delta_pct`   = total_delta_usd / total_prior_usd
- `untagged_pct_current` = W-cost.untagged_pct (TAG mode only; N/A — omit the
  Chargeback Coverage row — in dimensional mode)
- `top_owner` = W-cost top owner (dimensional cell/region, or tag value)
- `biggest_grower` = W-movers.top_positive_movers[0]
- `biggest_saver` = W-movers.top_negative_movers[0]

---

### Phase 3: Davis CoPilot Synthesis (OPTIONAL)

**Only invoke Davis CoPilot if the cost data is rich.** A clean cost report
with a clear top owner, a single dominant mover, and no anomalies often
doesn't need CoPilot — the attribution narrative speaks for itself.

**INVOKE Davis CoPilot when:**
- `total_delta_pct` exceeds ±15% (material WoW or MoM change → ask for the why).
- `untagged_pct_current` > 20% (significant chargeback gap → ask for remediation).
- Three or more positive movers each exceeding 20% growth (multi-initiative
  ramp; want a categorization).
- A dropped tag value previously accounted for > 10% of total spend (large
  decommission; want a sanity-check on whether it's intentional).

**SKIP Davis CoPilot when:**
- Total spend is flat (±5%) and top owner is stable. The report writes itself.
- No new or dropped tag values. No movers. Pure steady-state.

When invoking:
```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
COST ATTRIBUTION: {CLOUD} | {SCOPE} | ATTRIBUTION={ATTRIBUTION_MODE}{tag mode: " key={TAG_KEY}"}
PERIOD: {CURRENT_PERIOD.label} (vs prior {PRIOR_PERIOD.label})

TOTAL: ${total_current_usd} ({delta_pct}% vs prior)
TOP OWNER: {top_owner} (${top_owner.cost_usd}, {pct_of_total}%)   [owner = dimensional cell/region, or tag value]

TOP POSITIVE MOVERS:
  {mover_key}: +${delta_usd} (+{delta_pct}%)
  ...
TOP NEGATIVE MOVERS:
  {mover_key}: −${delta_usd} ({delta_pct}%)
  ...
NEW INITIATIVES:   {new_keys list}
DECOMMISSIONED:    {dropped_keys list}
UNTAGGED SPEND:    {ATTRIBUTION_MODE == "tag" ? "{untagged_pct_current}%" : "N/A (dimensional mode)"}

QUESTIONS:
1. Does the spend distribution suggest concentration risk or healthy diversity?
2. For each top positive mover, is the growth pattern (linear, step-change,
   spike) consistent with a planned initiative or an anomaly?
3. Recommend three concrete FinOps optimizations for the top owner.
4. If untagged spend > 20%, recommend a tagging remediation approach.
```

Davis CoPilot response feeds the "Cost Narrative" and "Recommended
Optimizations" sections. If skipped, those sections are written deterministically
from the worker data.

---

### Phase 4: Report Generation

If CLEAN_MODE, build the sanitization map first (identical to dt-rca Phase 1.14
— read that section for full rules) and compose with sanitized names from the
start. Sanitize: account IDs, subscription UUIDs, project IDs, tag values that
contain company/team/owner identifiers, resource names, tenant URL.

**Emit the report EXACTLY in the section order below.** Each fact has ONE
canonical home; never restate the same data in another section.

> **OWNER COLUMN is mode-dependent.** Where the template shows `{OWNER_LABEL}`:
> in TAG mode it is `{TAG_KEY}`; in DIMENSIONAL mode it is the attribution axis,
> e.g. `Region / Category / Instance Type` for the canonical table and
> `Instance Type` for the movers table. The title line shows `TAG: {TAG_KEY}`
> only in tag mode; in dimensional mode it shows `Attribution: dimensional`.

```markdown
# dt-cloud-cost FinOps Attribution Report
## {CLOUD} / {SCOPE} / {TAG mode: "TAG: {TAG_KEY}" | dimensional: "Attribution: dimensional"} — {CURRENT_PERIOD.label}

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Cloud:** {CLOUD}  |  **Scope:** {SCOPE or "(full {CLOUD})"} ({SCOPE_KIND})  |  **Attribution:** {ATTRIBUTION_MODE}{tag mode: " (key {TAG_KEY})"}  |  **Cost Source:** {COST_EVENT_TYPE} → {COST_FIELD}
**Current Period:** {CURRENT_PERIOD.start} → {CURRENT_PERIOD.end}
**Prior Period:**   {PRIOR_PERIOD.start} → {PRIOR_PERIOD.end}

{IF ATTRIBUTION_MODE=="dimensional" AND TAG_KEY was supplied AND tag_join_empty:
> **Note:** A TAG (`{TAG_KEY}`) was requested, but `dt.entity.host` is not
> populated on this tenant's cost events (the host→tag join returned 0 rows), so
> spend is attributed by native event dimensions instead. To enable tag-based
> chargeback, ensure cost events carry `dt.entity.host` and that hosts carry the
> `{TAG_KEY}` tag.}

---

## Executive Summary

### Headline Metrics
| Metric | Value | vs Prior |
|--------|-------|----------|
| Total Spend (current) | ${total_current_usd} | {total_delta_pct}% |
| Total Spend (prior)   | ${total_prior_usd}   | — |
| Top Owner             | {top_owner} (${top_owner.cost_usd}, {pct}%) | |
| Biggest Grower        | {biggest_grower.mover_key} (+${delta_usd}) | +{delta_pct}% |
| Biggest Saver         | {biggest_saver.mover_key} (−${delta_usd}) | {delta_pct}% |
{IF ATTRIBUTION_MODE=="tag": | Chargeback Coverage   | {100-untagged_pct}% tagged | — | }

### Narrative
[2–3 sentence prose synthesis from Phase 2: who owns spend, what changed,
what's new, what dropped off. Lead with the headline. Cite Davis CoPilot if
invoked.]

### Immediate Action Required
| Priority | Action | Owner | Urgency |
|----------|--------|-------|---------|
[Top 1–3 recommendations — derived from Davis CoPilot if invoked, otherwise
deterministic: review biggest grower, validate new initiatives, address
untagged spend if >20%.]

### Spend Trend Visualization
```mermaid
gantt
    title Spend Distribution: {SCOPE} ({OWNER_LABEL})
    [Stacked-area-style timeline showing top-5 owners across both periods.
     Per dt-rca Mermaid rules — INLINE here; do not repeat elsewhere.]
```

---

## Cost Attribution Table

The canonical owner × spend × delta table. Every other section references THIS.
In dimensional mode the owner key is the (Region / Category / Instance Type)
cell from W-cost's `dimensional_rollup`; the `(untagged)` row is OMITTED. In tag
mode the owner key is `{TAG_KEY}` and the `(untagged)` row is INCLUDED.

| Rank | {OWNER_LABEL} | Current ($) | Prior ($) | Δ ($) | Δ (%) | % of Total |
|------|---------------|-------------|-----------|-------|-------|------------|
| 1    | ...           | ...         | ...       | ...   | ...   | ...        |
| ...  | ...           | ...         | ...       | ...   | ...   | ...        |
| —    | (untagged)*   | ...         | ...       | ...   | ...   | ...        |
| **Total** | | ${total_current_usd} | ${total_prior_usd} | ${total_delta_usd} | {total_delta_pct}% | 100% |

\* TAG mode only — omit in dimensional mode.

---

## Top Movers

### Positive Movers (Growth)
| {OWNER_LABEL} | Prior ($) | Current ($) | Δ ($) | Δ (%) |
|---------------|-----------|-------------|-------|-------|
[Top 10 from W-movers.top_positive_movers — `mover_key` is the instance-type
(dimensional) or tag value (tag)]

### Negative Movers (Savings)
| {OWNER_LABEL} | Prior ($) | Current ($) | Δ ($) | Δ (%) |
|---------------|-----------|-------------|-------|-------|
[Top 10 from W-movers.top_negative_movers]

---

## Per-Owner Resource Detail

For each of the top 5 owners (dimensional: regions; tag: tag values), what are
they paying for?

### Owner: {owner_1}  ({"region" in dimensional mode | TAG_KEY in tag mode})
**Total this period:** ${total_usd}

| Resource Category | Instance Type | Cost ($) |
|-------------------|---------------|----------|
[From W-attribution.per_owner[0].resource_category_breakdown]

**Top 5 hosts:** (TAG mode only — omit in dimensional mode, where host is NULL)
| Host ID | Host Name | Cost ($) |
|---------|-----------|----------|
[From W-attribution.per_owner[0].top_hosts]

[Repeat the above block for the remaining 4 top owners.]

---

## New & Dropped Initiatives

### New This Period
Owner keys present in {CURRENT_PERIOD.label} but absent in {PRIOR_PERIOD.label}.
Often signals new initiatives, first-seen instance types, projects, or teams.

| {OWNER_LABEL} | Current Spend ($) |
|---------------|--------------------|
[From W-movers.new_keys]

### Dropped This Period
Owner keys present in {PRIOR_PERIOD.label} but absent in {CURRENT_PERIOD.label}.
Often signals decommissioned workloads, retired instance types, or sunset projects.

| {OWNER_LABEL} | Prior Spend ($) |
|---------------|------------------|
[From W-movers.dropped_keys]

---

## Recommended Optimizations

[From Davis CoPilot if invoked; otherwise deterministic recommendations:]
1. **Review the biggest grower.** {biggest_grower.mover_key} grew
   {biggest_grower.delta_pct}% — confirm planned vs anomalous; consider Reserved
   Instances / Savings Plans / Committed-Use Discounts if growth is sustained.
2. **Validate new initiatives.** Confirm budget owners for each new owner key;
   add a budget alert if missing.
3. **(TAG mode only) Address untagged spend.** If chargeback coverage is below
   80%, mandate the {TAG_KEY} tag on resource creation via policy (AWS SCP /
   Azure Policy / GCP Organization Policy). Untagged spend currently runs
   ${untagged_cost_usd} ({untagged_pct}% of total).
   **(DIMENSIONAL mode) Enable chargeback.** Cost events lack `dt.entity.host`,
   so per-team chargeback is not yet possible — enable host attribution on the
   carbon-impact app and tag hosts to unlock tag-based showback.
4. **Idle-resource sweep for the top owner.** Cross-reference the top owner's
   resources against the cloud's idle-detection patterns (EC2 < 5% CPU, Lambda
   zero invocations, unattached disks, etc. — see
   `~/.claude/skills/dt-obs-{CLOUD}/references/cost-optimization.md`).

---

## Appendix A: Methodology

**Cloud filter:** `cloud.provider == "{CLOUD}"`
**Scope filter:** `{SCOPE_FILTER or "(none — full {CLOUD} rollup)"}`
**Attribution mode:** {ATTRIBUTION_MODE}
  - dimensional: native event dimensions `cloud.region` / `resource.category` /
    `resource.instance.type` (cost events carry no populated `dt.entity.host` on
    this tenant, so the host→tag join is not used).
  - tag: {TAG_KEY} (resolved via Smartscape HOST `tags[]` join on `dt.entity.host`).
**Probe result:** {PROBE_SUMMARY}
**Cost source:** `event.provider == "{EVENT_PROVIDER}"` /
`event.type == "{COST_EVENT_TYPE}"` / field `{COST_FIELD}` (currency `{COST_CURRENCY_FIELD}`)
**Cost mode:** {COST_EVENT_TYPE == "cost.list.price" ? "list price (catalog)" : "realized spend"}
**Current period:** [{CURRENT_PERIOD.start}, {CURRENT_PERIOD.end})
**Prior period:**   [{PRIOR_PERIOD.start},   {PRIOR_PERIOD.end})
**Sub-skills consumed:** dt-obs-{CLOUD} (`resource-ownership.md`,
`cost-optimization.md`)
**Worker count:** 4 (W-cost, W-baseline, W-movers, W-attribution) dispatched in
ONE parallel batch.
**Davis CoPilot:** {invoked: yes/no — reason}.
**Sanitization:** {CLEAN_MODE: yes/no}.

**Constraints (v1):** Cost data is sourced exclusively from the Dynatrace
carbon-impact app bizevents (`event.provider == "dynatrace.biz.carbon"`).
Other ingestion shapes — direct AWS CUR exports, Azure Cost Management
exports, GCP Cloud Billing exports — are NOT supported by v1 and require
their own extractor. The carbon-app bizevents do not carry native cloud
account IDs / subscription UUIDs / project IDs, so SCOPE is restricted to
`cloud.region` or `dt.entity.host`.

[Optional: Appendix B Glossary + Appendix C Capabilities if `appendix=true`
is set — same expansion as /dt-rcf.]

---

**Links:**
- View tenant: {TENANT_URL}

---

*End of Report*
```

---

### Phase 5: Save Report (Markdown)

**The Markdown is the canonical deliverable. Always write the `.md`.**

```
If CLEAN_MODE = false:
  Filename: Cost_{CLOUD}_{SCOPE_SLUG}_{CURRENT_PERIOD.label}_Report.md
If CLEAN_MODE = true:
  Filename: Cost_{CLOUD}_{SANITIZED_SCOPE_SLUG}_{CURRENT_PERIOD.label}_Report_SANITIZED.md
```

`SCOPE_SLUG` = `"all"` when SCOPE_KIND=="none"; the region value when "region";
the HOST id (full) when "host".
**Location:** current working directory.

---

### Phase 6: Generate PDF (OPT-IN — only if `PDF_MODE == true`)

**DEFAULT IS MARKDOWN-ONLY (Mandate 1).** Skip this phase entirely unless
`--pdf` was passed. The Markdown from Phase 5 is the complete deliverable.

If `PDF_MODE == true`: identical to `/dt-rca` Phase 4 — create a `_pdf.md`
intermediate, replace ALL Mermaid blocks with ASCII art per the dt-rca rules,
convert with `md-to-pdf`, rename, delete the intermediate. **Wrap the conversion
so a missing engine is non-fatal:**

```bash
if command -v md-to-pdf >/dev/null 2>&1; then
  md-to-pdf Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_pdf.md \
    --pdf-options '{"format": "A4", "margin": {"top": "15mm", "bottom": "15mm", "left": "12mm", "right": "12mm"}, "printBackground": true}'
  mv Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_pdf.pdf Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_Report.pdf
  rm -f Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_pdf.md
else
  echo "PDF engine unavailable — Markdown only"
fi
```

A missing `md-to-pdf` engine logs exactly one line (`PDF engine unavailable —
Markdown only`) and the run continues successfully. **Never hard-fail a run on
PDF.** (Replace filename suffixes with `_SANITIZED` in CLEAN_MODE.)

---

### Phase 7: Output Summary

```markdown
## FinOps Report Generated

### Files Created
| Format | Filename |
|--------|----------|
| Markdown | Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_Report[_SANITIZED].md |
| PDF | {PDF_MODE ? "Cost_{CLOUD}_{SCOPE_SLUG}_{PERIOD}_Report[_SANITIZED].pdf" : "skipped (MD-only default; pass --pdf to enable)"} |
| Run log | COST_{CLOUD}_{SCOPE_SLUG}_{DATE}.log |

**Location:** {FULL_PATH}
**Sanitized:** {YES/NO}

### Quick Summary
- **Cloud:** {CLOUD}
- **Scope:** {SCOPE or "(full {CLOUD})"} ({SCOPE_KIND})
- **Cost source:** {COST_EVENT_TYPE} ({COST_EVENT_TYPE == "cost.list.price" ? "list price" : "realized spend"})
- **Attribution:** {ATTRIBUTION_MODE}{tag mode: " (key {TAG_KEY})"}
- **Total spend (current):** ${total_current_usd} ({total_delta_pct}% vs prior)
- **Top owner:** {top_owner} ({pct_of_total}%)
- **Biggest grower:** {biggest_grower.mover_key} (+${delta_usd})
- **New initiatives:** {count}
- **Decommissioned:** {count}
- **Untagged spend:** {ATTRIBUTION_MODE == "tag" ? "{untagged_pct}%" : "N/A (dimensional mode)"}

**View options:** Markdown via `Ctrl+Shift+V` in VS Code; pass `--pdf` to also
generate a PDF for distribution.
```

If CLEAN_MODE, also print the sanitization mapping table to the console
(NEVER to a file).

---

## QUALITY RULES (NON-NEGOTIABLE)

1. **BILLING-EMPTY GATE IS NON-NEGOTIABLE.** If
   `fetch bizevents | filter event.provider == "dynatrace.biz.carbon" | filter
   cloud.provider == "{CLOUD}" {SCOPE_FILTER}` returns 0 events for the current
   period, FAIL FAST with the carbon-app onboarding message. Never silently
   produce a USD 0 report.
2. **SCHEMA DISCOVERY OVER GUESSING.** If a cost query errors
   FIELD_DOES_NOT_EXIST or UNKNOWN_DATA_OBJECT, the worker runs ONE
   `fetch bizevents | filter event.provider == "dynatrace.biz.carbon" |
   summarize count(), by: { event.type } | limit 20` probe and records the
   result in `field_discovery`. Never substitute `billing.cost.usd`,
   `fetch billing`, or any `{cloud}.billing` event type — those do NOT exist
   in v1's substrate.
3. **SCOPE CLASSIFICATION HAPPENS BEFORE ANY QUERY.** SCOPE starting with
   `HOST-` → host filter; otherwise → region filter; absent → no scope filter.
   No native account/subscription/project IDs in v1.
4. **ALL FOUR WORKERS DISPATCH IN ONE MESSAGE.** Sequential dispatch defeats the
   architecture. If you find yourself sending a second Agent call after waiting
   for the first to return, stop and fix it.
5. **REFERENCE-DRIVEN DQL ONLY.** Workers read the cloud's
   `resource-ownership.md` + `cost-optimization.md` and derive queries from those
   patterns. No DQL from training knowledge. Drift is impossible by design.
6. **TAGGED + UNTAGGED MUST SUM TO TOTAL (TAG mode only).** When
   ATTRIBUTION_MODE=="tag", if `tagged_cost_usd + untagged_cost_usd !=
   total_cost_usd` within a USD 1 tolerance, the worker has a bug — re-dispatch
   with a note to verify the filter logic. In dimensional mode, instead check
   that the sum of `dimensional_rollup` cost ≈ `total_cost_usd` (top-20 cap may
   truncate the tail; the sanity total query is authoritative).
7. **CLEAN MODE = ZERO LEAKS.** Sanitize account IDs, subscription UUIDs,
   project IDs, tag values containing identifying names (Owner emails, Team
   names, Project codes), resource IDs, tenant URL. Scan the final report
   before saving.
8. **DAVIS COPILOT IS OPTIONAL.** Invoke only when the data is rich (see
   Phase 3 triggers). A flat steady-state report does not need CoPilot.
9. **MERMAID FOR .MD, ASCII FOR PDF.** Same two-file strategy as `/dt-rca`,
   applied ONLY when PDF_MODE is on. Raw Mermaid in PDFs is forbidden.
10. **DIMENSIONAL REPORTS ARE FIRST-CLASS — NEVER ABORT ON A MISSING TAG (CC-0/CC-2).**
    A missing TAG, or a TAG whose host→tag join resolves zero rows, is NOT a
    failure: the skill produces a DIMENSIONAL report (by `cloud.region` /
    `resource.category` / `resource.instance.type`) instead. Only abort if the
    **Billing-Empty Gate** fires (no cost events at all). If a TAG was requested
    but could not be applied, add the one-line note in the report header
    explaining the fallback — do NOT print "chargeback impossible" and exit. The
    ONLY hard-abort in this skill is Rule 1 (zero cost events).

---

## ERROR HANDLING

### If SCOPE not found

```markdown
## Error: Scope Not Present in Carbon-App Bizevents

`CLOUD={CLOUD}` SCOPE=`{SCOPE}` ({SCOPE_KIND}) returned zero carbon-app bizevents
in {CURRENT_PERIOD.label}.

**Possible causes:**
- SCOPE value is incorrect (typo, wrong region code, wrong HOST id format)
- The carbon-impact app is not ingesting for this region / host
- The host has not produced cost events in the requested period

**Available actions:**
- List regions present for {CLOUD}:
  ```
  fetch bizevents, from:now()-7d
  | filter event.provider == "dynatrace.biz.carbon"
  | filter cloud.provider == "{CLOUD}"
  | summarize n = count(), by: { cloud.region }
  | sort n desc
  ```
- List hosts with carbon-app cost in the period:
  ```
  fetch bizevents, from:toTimestamp("{CURRENT_PERIOD.start}"), to:toTimestamp("{CURRENT_PERIOD.end}")
  | filter event.provider == "dynatrace.biz.carbon"
  | filter cloud.provider == "{CLOUD}"
  | summarize cost = sum({COST_FIELD}), by: { dt.entity.host }
  | sort cost desc | limit 20
  ```
```

### If billing ingestion missing

See the Phase 1.5 Billing-Empty Gate message — that is the canonical text.

### If TAG cannot be applied (host join empty / tag absent) — DEGRADE, do NOT exit

This is **not an error** (CC-0/CC-2). When ATTRIBUTION_MODE could not become
"tag" — because `dt.entity.host` is NULL on cost events, or the requested tag is
absent from all hosts — the skill produces a DIMENSIONAL report and adds the
header note explaining the fallback. The block below is **diagnostic guidance to
embed in that note**, not an exit path. Only the Billing-Empty Gate exits.

```markdown
## Note: Tag Attribution Unavailable — Reported by Native Dimensions Instead

Tag `{TAG_KEY}` could not be applied: {reason — "cost events carry no
dt.entity.host on this tenant" | "the tag is absent from all monitored hosts"}.
Spend is attributed by `cloud.region` / `resource.category` /
`resource.instance.type` instead. The totals are exact; only the per-team
chargeback breakdown is unavailable until host attribution + tagging are enabled.

**To enable tag-based chargeback later:**
- Discover commonly-used tags on monitored hosts:
  ```
  smartscapeNodes "HOST"
  | filter isNotNull(tags)
  | expand tags
  | summarize host_count = countDistinct(id), by: { tag_key = tags[0] }
  | sort host_count desc | limit 30
  ```
- Tag-coverage audit for the requested key:
  ```
  smartscapeNodes "HOST"
  | fieldsAdd has_tag = if(isNotNull(tags[`{TAG_KEY}`]), 1)
  | summarize total = count(), with_tag = sum(has_tag)
  | fieldsAdd coverage_pct = (with_tag * 100.0) / total
  ```
- Reference docs:
  - AWS:   `~/.claude/skills/dt-obs-aws/references/resource-ownership.md`
  - Azure: `~/.claude/skills/dt-obs-azure/references/resource-ownership.md`
  - GCP:   `~/.claude/skills/dt-obs-gcp/references/resource-ownership.md`
```

### If worker queries fail

Re-dispatch the ONE failing worker once. If it fails again, record `⚠️ gap` in
the report and continue. Do not run worker queries inline in the orchestrator
thread.

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0-auth. Execute all phases autonomously
through Phase 7.

**Do not stop for confirmation. Do not ask questions. Generate the complete
report — unless the Billing-Empty Gate fires, in which case fail fast with the
ingestion-required message.**
