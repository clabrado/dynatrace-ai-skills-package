# Dynatrace Uber-Skills — Full Reference

**Version:** v3 (tenant-validated 2026-05-28; re-validated end-to-end + corrected 2026-06-03)  
**Author:** Chris LaBrado (Lead Solutions Engineer, Dynatrace)  
**Substrate authority:** `dt-rcf` SKILL.md — all uber-skills inherit its phased execution model

This document is the authoritative reference for all Tier 1 and Tier 2 skills in this package. The SKILL.md files in `skills/` are the executable versions; this document provides design rationale, real-world use cases, and tenant-validated substrate notes.

---

## ⚠️ 2026-06-03 UPDATE — read this first

All 8 Tier 1/2 skills were re-validated end-to-end on 2026-06-03 and corrected. The headline changes below supersede the original 2026-05-28 text wherever they conflict:

- **Output is Markdown-only by default.** Every artifact described below ships as Markdown; PDF is opt-in via `--pdf` and non-fatal — a missing `md-to-pdf` engine logs one line and the `.md` still ships.
- **Probe-then-query (Phase 0b.0 Substrate Probe).** Each skill now discovers the live `event.kind` / field / vocabulary at runtime (candidate list `[corrected, legacy]`) instead of hardcoding a snapshot, so it self-heals against field and event-kind drift.
- **Per-skill substrate corrections:**
  - **dt-vuln-blast** — CVEs match via `in(CVE, vulnerability.references.cve)` (array), `event.kind == "SECURITY_EVENT"`, affected entities in `affected_entity.id` (`PROCESS_GROUP-*`).
  - **dt-deploy-risk** — deploys are `SDLC_EVENT` / `task.deployment.finished`; a `component` → `service` bridge resolves the missing service link.
  - **dt-cloud-cost** — `TAG:` is optional, default cost feed is `cost.list.price`, and dimensional attribution (region / category / instance-type) is primary.
  - **dt-ai-obs** — `APP:` resolves as a SERVICE or as a `gen_ai.request.model` value; self-hosted / unpriced models price at $0 so an all-self-hosted tenant still renders a full brief.
  - **dt-dem-vitals** — vitals timeseries require an explicit `interval:`; `BASELINE:`/`COMPARE:` are single instants.
  - **dt-k8s-podf** — auto-widens 6h → 72h when the default window is empty; cause-class counters read Davis `event.type` (CUSTOM_INFO / ERROR_EVENT / WARNING).
  - **dt-rum-journey** — a live-vocabulary probe gates absent funnel steps (only `click` is a `has_user_action` interaction tenant-wide).
- **Anti-hallucination guardrails.** A pre-dispatch reality gate refuses to fan out workers if Phase 0b couldn't run a successful query; every worker prompt opens with "no data = `<gap>`, never fabricate a number."
- **Auth hardened.** Workers never run `dtctl auth login`/`refresh` — auth is orchestrator-only, interactive, and never races concurrent logins (a concurrent-login race corrupted the token store during testing).

Full re-test log: `dt-uber-skills-fixrun-results-2026-06-03.md`.

---

## What's an uber-skill?

An uber-skill takes one anchored input (a CVE, an SLO, a deploy event, a pod, etc.), dispatches parallel reference-driven workers across composed base skills (`dt-obs-*`), synthesizes the findings (often via Davis CoPilot), and emits a polished artifact — typically Markdown (PDF optional via `--pdf`), sometimes a deployable Dynatrace Notebook or Dashboard.

```
ANCHOR ─▶ Phase 0a parse ─▶ 0-auth refresh ─▶ 0b bootstrap ─▶ 0b.0 substrate probe ─▶ 0c reference strategy
         ─▶ Phase 1 parallel workers (ALL in ONE message)
         ─▶ 1.5 absence-gate
         ─▶ Phase 2 synthesis (hypothesis graph / scorecard)
         ─▶ Phase 3 Davis CoPilot
         ─▶ Phase 4 report (Markdown; PDF optional via --pdf) or notebook/dashboard deploy
```

All Tier-1/2 uber-skills support `-clean` for sanitized customer-shareable output (sanitization map per `dt-rca` Phase 1.14).

---

## Quick Index

### Incident-anchored

| Skill | Anchor | Artifact |
|---|---|---|
| `/dt-rca` | Problem ID | Executive RCA (Markdown; PDF optional) |
| `/dt-rca-swarm` | Problem ID | Multi-agent RCA + reviewer (Markdown; PDF optional) |
| `/dt-rcf` | P / SERVICE / TIME / TRACE / ERROR | Forensic investigation (Markdown; PDF optional) |
| `/dt-pr-notebooks` | Problem ID | Deployed Notebook |
| `/dt-pr-dashboard` | Problem ID | Deployed Dashboard |

### Tier 1 — proactive / pre-emptive (validated)

| Skill | Anchor | Artifact | Substrate status |
|---|---|---|---|
| `/dt-slo-burn` | SLO ID or name | Burn briefing (Markdown; PDF optional) | ✅ Validated end-to-end |
| `/dt-vuln-blast` | CVE or library@version | Blast radius (Markdown; PDF optional) + optional PR | ✅ Validated |
| `/dt-deploy-risk` | Deploy event or service+version | Risk scorecard (Markdown; PDF optional) | ✅ Validated |
| `/dt-dem-vitals` | RUM app + windows | Web Vitals regression (Markdown; PDF optional) | ✅ Validated |
| `/dt-k8s-podf` | NS + workload | Pod forensics (Markdown; PDF optional, or notebook) | ✅ Validated |

### Tier 2 — specialized (validated)

| Skill | Anchor | Artifact | Substrate status |
|---|---|---|---|
| `/dt-cloud-cost` | CLOUD + scope + tag | FinOps attribution (Markdown; PDF optional) | ✅ Validated ($152K Azure surfaced) |
| `/dt-rum-journey` | App + steps + windows | Funnel diff (Markdown; PDF optional) | ✅ Validated |
| `/dt-ai-obs` | LLM app + window + framework | LLM observability brief (Markdown; PDF optional) | ✅ Validated (58K spans surfaced) |

---

## Value Summary

| # | Skill | What it does | Net value add |
|---|---|---|---|
| 1 | `/dt-slo-burn` | Multi-window burn-rate (1h+5m fast, 6h+30m slow) on any Custom-SLI SLO; optional `--forecast` projects budget exhaustion. Verdict ladder GO / SLOW / FAST / FREEZE. | Release-freeze decision in minutes. Wires into CI as a deploy gate. Replaces Slack debate with a sourced exec brief. |
| 2 | `/dt-vuln-blast` | CVE or library@version → prioritized fix list weighted by real span-graph reachability, traffic, internet exposure, ownership. Optional `--pr` drafts a remediation PR. | AppSec triage by exploitability, not just CVSS. Hours of SBOM cross-ref collapses to one report. |
| 3 | `/dt-deploy-risk` | Pre vs post-deploy delta across RED metrics, new exceptions, log patterns, dep churn. Weighted score 0-100 → GO / HOLD / ROLLBACK. | Defensible go/rollback before the next page fires. Self-service post-deploy check for engineers. |
| 4 | `/dt-dem-vitals` | Web Vitals (INP/LCP/CLS/FCP/TTFB) regression diff between two windows or auto-derived around a deploy. Classifies as frontend / backend / network with span attribution. | Ends frontend-vs-backend finger-pointing on UX regressions. |
| 5 | `/dt-k8s-podf` | "kubectl describe on steroids" — joins K8s events, logs, traces, network policies. Classifies cause (OOM / CrashLoop / ImagePull / probe / denied egress / DNS / scheduling / app). | 3am pod debug in 60s instead of an hour across kubectl/logs/traces. |
| 6 | `/dt-cloud-cost` | AWS/Azure/GCP cost rollup from carbon-app bizevents by tag/region/host. Surfaces top movers + new initiatives + decommission candidates vs prior period. | FinOps monthly review without leaving Dynatrace. |
| 7 | `/dt-rum-journey` | RUM funnel diff across N steps between two windows. Identifies the biggest leak step + the most-frequent next action users took instead + backend errors at that step. | "Where are users leaving — and why" in one shot. |
| 8 | `/dt-ai-obs` | Framework-aware (langgraph / crewai / openai / anthropic) brief over OpenLLMetry gen_ai spans. Per-prompt latency, token-cost outliers, agent-loop detection, $/feature. | LLM cost spike triage, loop debugging, model ROI comparison. |

---

## Tier 1 — Detailed Reference

### 1. `/dt-slo-burn` — Error-Budget Burn Briefing

**Syntax:**
```
/dt-slo-burn SLO:"<id-or-name>" [HORIZON:1h|6h|24h|30d] [--forecast] [-clean]
```

**Arguments:**

| Arg / flag | Default | Description |
|---|---|---|
| `SLO:"<id-or-name>"` | required | Opaque SLO ID or human name (resolved via dtctl get slo) |
| `HORIZON:1h\|6h\|24h\|30d` | `1h` | Evaluation window length |
| `--forecast` | off | Project budget-exhaustion datetime via predictive analytics |
| `-clean` | off | Sanitize per dt-rca Phase 1.14 |

**Design intent:** Closes the gap between "Davis told me there's a problem" and "should I FREEZE the release pipeline." Computes multi-window burn-rates per Google SRE Workbook ch. 5 (fast 1h+5m @ 14.4×, slow 6h+30m @ 6×). Verdict ladder: **GO** / **SLOW BURN** / **FAST BURN/FREEZE** / **NO TRAFFIC**.

**Composed sub-skills:** `dt-obs-problems` + `dt-obs-services` + `dt-obs-predictive-analytics` (forecast only)

**Real-world use cases:**
1. **Pre-release gate** — "Should we ship 2.3.0 in the next 2 hours?" Run `--forecast`; if exhaustion is projected within the release window, FREEZE.
2. **Mid-incident exec brief** — during an active Davis problem, pull a one-page burn picture leadership can read.
3. **Weekly SRE review** — sweep through top SLOs at `HORIZON:30d`, capture trending burn.
4. **Adaptive freeze trigger** — wire into CI; verdict ≠ GO blocks the deploy stage.
5. **Customer escalation defense** — customer claims breach; pull a -clean briefing with sourced numbers.

**Substrate notes (validated 2026-05-28; re-validated 2026-06-03):**
- SLOs are settings records, not Grail data objects — accessed via `dtctl get slo` + `dtctl describe slo <id>` (NOT `fetch dt.slo`)
- Indicator DQL executed via `--default-timeframe-start/--default-timeframe-end` flags — never mutated inline
- v1: Custom-SLI SLOs only. Template-based SLOs fail fast with a clear message

---

### 2. `/dt-vuln-blast` — Vulnerability Blast Radius

**Syntax:**
```
/dt-vuln-blast {CVE-YYYY-NNNN|LIB:"name@version"} [-clean] [--pr REPO:"owner/repo"] [--apply]
```

**Design intent:** Turns a vulnerability ID into a prioritized fix list weighted by *real reachability* (call paths from the span graph), not just "library is installed." Maps affected services → workloads → owners → fix prioritization scorecard (reachability × traffic × internet-exposure × error-path involvement).

**Composed sub-skills:** `security.events` (AppSec) + `dt-obs-services` + `dt-obs-tracing` + `dt-obs-kubernetes`

**Substrate notes:** Data source is `security.events` (no `dt.` prefix). CVEs are matched via `in(CVE, vulnerability.references.cve)` (the CVE lives in an array, not a scalar `vulnerability.id`), records filter `event.kind == "SECURITY_EVENT"`, and affected entities are in `affected_entity.id` (`PROCESS_GROUP-*`, scoped via `dt.entity.process_group`). `--apply` gate enforced at parse time AND pre-write.

---

### 3. `/dt-deploy-risk` — Deployment Risk Scorecard

**Syntax:**
```
/dt-deploy-risk {DEPLOY:<event-id>|SERVICE:"<name>" VERSION:"<v>"} [--baseline 7d] [-clean]
```

**Design intent:** Replaces "wait for the next page" with a defensible deploy/rollback decision in minutes. Scores 0-100 via weighted rubric (error 0.35 / latency 0.25 / new exceptions 0.25 / dep churn 0.15) → **GO** (≥80) / **HOLD** (50-79) / **ROLLBACK** (<50).

**Composed sub-skills:** `dt-obs-services` + `dt-obs-tracing` + `dt-obs-problems` + `dt-obs-logs`

**Substrate notes:** 48h ambiguity fail-fast on SERVICE+VERSION fallback. Deploys are `SDLC_EVENT` / `task.deployment.finished` (CUSTOM_DEPLOYMENT is gone; legacy deployments now arrive under `DAVIS_EVENT`). `SDLC_EVENT` carries no service link → a `component.name` → SERVICE bridge (via spans) resolves it.

---

### 4. `/dt-dem-vitals` — Web Vitals Regression Brief

**Syntax:**
```
/dt-dem-vitals APP:"<rum-app>" {BASELINE:<iso> COMPARE:<iso>|--vs-deploy DEPLOY-ID} [-clean]
```

**Design intent:** Closes the loop between RUM Web Vitals and the *backend cause* of regressions. Frontend / backend / network classification is the synthesis output.

**Composed sub-skills:** `dt-obs-frontends` (AdvancedPerformance, RequestPerformance, RequestTimingAnalysis) + `dt-obs-tracing` (TraceCorrelation)

**Substrate notes:** Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events. Vitals `timeseries percentile(...)` requires an explicit `interval:` (default `1h`) — without it the auto-interval over-buckets and can collapse to 0 records over short windows. `BASELINE:<iso>` / `COMPARE:<iso>` are single instants (each defines `[point .. point + WINDOW_LEN]`). Degrades gracefully when metric dimensions are absent (tenant-aggregate scope instead of per-app).

---

### 5. `/dt-k8s-podf` — Pod Debug Forensics

**Syntax:**
```
/dt-k8s-podf NS:"<namespace>" WORKLOAD:"<workload>" [POD:"<pod-name>"] [--notebook] [-clean]
```

**Design intent:** "kubectl describe on steroids" — joins K8s events, container logs, distributed traces, AND applied network policies. Classifies root cause into 10+ classes. The network policy join catches "phantom errors" that other tools miss.

**Composed sub-skills:** `dt-obs-kubernetes` + `dt-obs-logs` + `dt-obs-tracing`. With `--notebook`: also `dt-app-notebooks`.

**Substrate notes:** K8s events surface as `event.kind == "DAVIS_EVENT"` filtered by `k8s.namespace.name` — NO `dt.system.bucket == "default_events"` filter. Auto-widens the window 6h → 72h when the default window returns 0 events (prevents a false `inconclusive`). The kubelet `reason` field is NULL on this tenant family → cause-class counters read `event.name` (e.g. "Back-off restarting") plus Davis `event.type` (CUSTOM_INFO / ERROR_EVENT / WARNING), not `reason` alone.

---

## Tier 2 — Detailed Reference

### 6. `/dt-cloud-cost` — Cloud Cost Attribution

**Syntax:**
```
/dt-cloud-cost CLOUD:aws|azure|gcp [SCOPE:region:<r>|host:<host-id>] [TAG:<key>] [--list-price] [-clean]
```

**Design intent:** Monthly FinOps attribution report without leaving Dynatrace. Pulls Grail bizevents from the carbon-impact app, sums cost by tag/team/region, computes deltas vs prior period.

**Composed sub-skills:** `dt-obs-aws` / `dt-obs-azure` / `dt-obs-gcp` (cost-optimization references) + `dynatrace.biz.carbon` bizevents

**Substrate notes:** `fetch billing` does NOT exist. Source is `bizevents` with `event.provider == "dynatrace.biz.carbon"`. `TAG:` is OPTIONAL (bare `CLOUD:azure` runs in dimensional mode — no required-tag abort). Default cost feed is `cost.list.price` / `price.total` (the populated feed; `cost.list.spend` is near-empty). `dt.entity.host` is NULL on cost events → dimensional attribution (region / resource-category / instance-type) is the primary rollup and the host→tag join is an optional overlay. **Tenant validated:** $152K Azure (June MTD) — westeurope $57K / eastus $50K / eastus2 $45K.

---

### 7. `/dt-rum-journey` — User Journey Funnel Diff

**Syntax:**
```
/dt-rum-journey APP:"<rum-app>" STEPS:"step1>step2>step3" {BASELINE:<iso> COMPARE:<iso>|--vs-deploy DEPLOY-ID} [-clean]
```

**Design intent:** Funnel conversion delta with backend attribution. Identifies the largest drop-off, names the most-frequent NEXT action users took instead of progressing, and correlates to backend errors.

**Composed sub-skills:** `dt-obs-frontends` (UserAction, NavigationPatterns, user-sessions) + `dt-obs-tracing` (TraceCorrelation)

**Substrate notes:** Data source is `user.events`. UserAction matching on `interaction.name` exclusively — `user_action.name` has zero rows on most tenant families. A Phase 0b.0 live-vocabulary probe surfaces the actual `interaction.name` / `page.url.path` values and gates absent steps: only `click` is flagged `has_user_action == true` tenant-wide, so `change`/`scroll` steps exist as interactions but are correctly reported absent (not phantom funnel leaks).

---

### 8. `/dt-ai-obs` — LLM Application Observability Brief

**Syntax:**
```
/dt-ai-obs APP:"<llm-app>" FROM:<iso> TO:<iso> --framework langgraph|crewai|openai|anthropic [-clean]
```

**Design intent:** Framework-aware analysis of GenAI/LLM apps instrumented with OpenLLMetry (`gen_ai.*` spans). Surfaces per-prompt latency, token-cost outliers, agent-loop detection, and per-feature unit economics.

**Composed sub-skills:** `dt-obs-tracing` (OpenLLMetry spans, request-attributes) + `dt-obs-services`

**Substrate notes:** gen_ai spans confirmed: 58,889 spans in 3d, p50=9ms p95=16ms on validation tenant. `APP:` resolves as a SERVICE **or** as a `gen_ai.request.model` value — resolve both and bind whichever matches (e.g. `genai-demo` binds as a model, `ai-travel-advisor-agent-test` as a service). The pricing table covers a self-hosted ($0) / unpriced class so an all-self-hosted tenant renders a full brief rather than collapsing to $0 and aborting. `gen_ai.system` is null/Azure (never the framework value) → warn-and-proceed. Framework-specific attributes: `langgraph.node`, `crewai.task.id`/`crewai.agent`, OTel `gen_ai.system`/`gen_ai.request.model`/`gen_ai.usage.*`.

---

## Shared Substrate

All uber-skills inherit:

| Phase | Behavior |
|---|---|
| 0-auth | `dtctl auth refresh` — orchestrator-only, interactive; workers NEVER run `auth login`/`refresh` (concurrent logins corrupt the token store) |
| 0a | Argument parser; flag table |
| 0b | Minimal sequential bootstrap (anchor resolution, 2-3 queries max) |
| 0b.0 | Substrate probe — discover the live `event.kind` / field / vocabulary at runtime (candidate list `[corrected, legacy]`); a pre-dispatch reality gate refuses to fan out if no query succeeded |
| 0c | Reference strategy — each worker reads ONE targeted reference file at start |
| 1 | Parallel workers — ALL dispatched in ONE message with multiple Agent calls; each worker treats "no data" as a `<gap>`, never a fabricated number |
| 1.5 | Absence-gate — any "not present" claim must be proven by unfiltered confirmation query |
| 2 | Synthesis — hypothesis graph (forensics) or scorecard (risk-scoring) |
| 3 | Davis CoPilot synthesis (where it adds value) |
| 4 | Report generation — Markdown by default (PDF opt-in via `--pdf`, non-fatal if `md-to-pdf` is absent); Mermaid in MD / ASCII in PDF |
| `-clean` | Sanitization per dt-rca Phase 1.14 |

---

## Tenant-Validated Lessons (2026-05-28; re-validated + extended 2026-06-03)

| # | Lesson |
|---|---|
| 1 | dtctl drops the `{ok, result}` envelope in subprocess — parse with `parsed.get("result", parsed)`. **(2026-06-03)** A third shape exists — the envelope key is sometimes `records` (not `result`/bare list); handle all three |
| 2 | `dt.slo`, `dt.security.events`, `dt.rum.web.events`, `dt.kubernetes.events`, `fetch billing` do NOT exist as Grail data objects |
| 3 | Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events. **(2026-06-03)** Windowed `timeseries percentile(...)` REQUIRES an explicit `interval:` — without it it mis-buckets / returns 0 over short windows |
| 4 | `dt.system.bucket == "default_events"` returns 0 on this tenant family — drop it; K8s events surface as `event.kind == "DAVIS_EVENT"`. **(2026-06-03)** The kubelet `reason` field is NULL — classify on `event.name` + Davis `event.type` (CUSTOM_INFO / ERROR_EVENT / WARNING) |
| 5 | Cost data is in `bizevents` with `event.provider == "dynatrace.biz.carbon"`. **(2026-06-03)** Default to the populated `cost.list.price` / `price.total` feed (spend is near-empty); `dt.entity.host` is NULL on cost events → attribute by native dimensions, not a host→tag join |
| 6 | Tenant metric dimensions can be absent — skills must degrade gracefully, not fail |
| 7 | UserAction names are on `interaction.name`, not `user_action.name`, on this tenant family. **(2026-06-03)** Only `click` is flagged `has_user_action == true` tenant-wide — probe the live vocabulary and gate absent steps |
| 8 | Use `--default-timeframe-start`/`--default-timeframe-end` on `dtctl query` to inject windows — never mutate the DQL string |
| 9 | **(2026-06-03)** AppSec: CVEs are in the `vulnerability.references.cve` ARRAY (match `in(CVE, …)`), `event.kind == "SECURITY_EVENT"`, affected entities in `affected_entity.id` (`PROCESS_GROUP-*`, scope via `dt.entity.process_group`) |
| 10 | **(2026-06-03)** Deploys are `SDLC_EVENT` / `task.deployment.finished` (CUSTOM_DEPLOYMENT is gone; legacy arrives under DAVIS_EVENT); SDLC_EVENT has no service link → bridge `component.name` → SERVICE via spans |
| 11 | **(2026-06-03)** gen_ai `APP` can be a SERVICE or a `gen_ai.request.model` VALUE — resolve both and bind whichever matches; price self-hosted / unknown models at $0 (never collapse the total to $0 and abort); `gen_ai.system` is null/Azure → warn-and-proceed |
| 12 | **(2026-06-03)** A Davis-problem membership filter on a dotted Smartscape field MUST use the function form `in("id", \`field\`)` — the infix form parse-errors |
| 13 | **(2026-06-03)** Never put a bare `$<digit>` in a SKILL.md — the harness expands `$0`/`$1`/… as positional-arg variables and corrupts the dollar literal (write `USD 0` / `USD 152K` instead); and workers must NEVER run `dtctl auth login`/`refresh` (racing non-interactive logins corrupts the token store) |
