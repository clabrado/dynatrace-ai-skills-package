# Dynatrace Uber-Skills — Full Reference

**Version:** v2 (tenant-validated 2026-05-28)  
**Author:** Chris LaBrado (Lead Solutions Engineer, Dynatrace)  
**Substrate authority:** `dt-rcf` SKILL.md — all uber-skills inherit its phased execution model

This document is the authoritative reference for all Tier 1 and Tier 2 skills in this package. The SKILL.md files in `skills/` are the executable versions; this document provides design rationale, real-world use cases, and tenant-validated substrate notes.

---

## What's an uber-skill?

An uber-skill takes one anchored input (a CVE, an SLO, a deploy event, a pod, etc.), dispatches parallel reference-driven workers across composed base skills (`dt-obs-*`), synthesizes the findings (often via Davis CoPilot), and emits a polished artifact — typically MD+PDF, sometimes a deployable Dynatrace Notebook or Dashboard.

```
ANCHOR ─▶ Phase 0a parse ─▶ 0-auth refresh ─▶ 0b bootstrap ─▶ 0c reference strategy
         ─▶ Phase 1 parallel workers (ALL in ONE message)
         ─▶ 1.5 absence-gate
         ─▶ Phase 2 synthesis (hypothesis graph / scorecard)
         ─▶ Phase 3 Davis CoPilot
         ─▶ Phase 4 report (MD + PDF) or notebook/dashboard deploy
```

All Tier-1/2 uber-skills support `-clean` for sanitized customer-shareable output (sanitization map per `dt-rca` Phase 1.14).

---

## Quick Index

### Incident-anchored

| Skill | Anchor | Artifact |
|---|---|---|
| `/dt-rca` | Problem ID | Executive RCA (MD+PDF) |
| `/dt-rca-swarm` | Problem ID | Multi-agent RCA + reviewer (MD+PDF) |
| `/dt-rcf` | P / SERVICE / TIME / TRACE / ERROR | Forensic investigation (MD+PDF) |
| `/dt-pr-notebooks` | Problem ID | Deployed Notebook |
| `/dt-pr-dashboard` | Problem ID | Deployed Dashboard |

### Tier 1 — proactive / pre-emptive (validated)

| Skill | Anchor | Artifact | Substrate status |
|---|---|---|---|
| `/dt-slo-burn` | SLO ID or name | Burn briefing (MD+PDF) | ✅ Validated end-to-end |
| `/dt-vuln-blast` | CVE or library@version | Blast radius (MD+PDF) + optional PR | ✅ Validated |
| `/dt-deploy-risk` | Deploy event or service+version | Risk scorecard (MD+PDF) | ✅ Validated |
| `/dt-dem-vitals` | RUM app + windows | Web Vitals regression (MD+PDF) | ✅ Validated |
| `/dt-k8s-podf` | NS + workload | Pod forensics (MD+PDF or notebook) | ✅ Validated |

### Tier 2 — specialized (validated)

| Skill | Anchor | Artifact | Substrate status |
|---|---|---|---|
| `/dt-cloud-cost` | CLOUD + scope + tag | FinOps attribution (MD+PDF) | ✅ Validated ($207K spend surfaced) |
| `/dt-rum-journey` | App + steps + windows | Funnel diff (MD+PDF) | ✅ Validated |
| `/dt-ai-obs` | LLM app + window + framework | LLM observability brief (MD+PDF) | ✅ Validated (58K spans surfaced) |

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

**Substrate notes (validated 2026-05-28):**
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

**Substrate notes:** Data source is `security.events` (no `dt.` prefix). Aggregate queries by `vulnerability.id` MUST `filter isNotNull(vulnerability.id)`. `--apply` gate enforced at parse time AND pre-write.

---

### 3. `/dt-deploy-risk` — Deployment Risk Scorecard

**Syntax:**
```
/dt-deploy-risk {DEPLOY:<event-id>|SERVICE:"<name>" VERSION:"<v>"} [--baseline 7d] [-clean]
```

**Design intent:** Replaces "wait for the next page" with a defensible deploy/rollback decision in minutes. Scores 0-100 via weighted rubric (error 0.35 / latency 0.25 / new exceptions 0.25 / dep churn 0.15) → **GO** (≥80) / **HOLD** (50-79) / **ROLLBACK** (<50).

**Composed sub-skills:** `dt-obs-services` + `dt-obs-tracing` + `dt-obs-problems` + `dt-obs-logs`

**Substrate notes:** 48h ambiguity fail-fast on SERVICE+VERSION fallback. `CUSTOM_DEPLOYMENT` events confirmed available.

---

### 4. `/dt-dem-vitals` — Web Vitals Regression Brief

**Syntax:**
```
/dt-dem-vitals APP:"<rum-app>" {BASELINE:<iso> COMPARE:<iso>|--vs-deploy DEPLOY-ID} [-clean]
```

**Design intent:** Closes the loop between RUM Web Vitals and the *backend cause* of regressions. Frontend / backend / network classification is the synthesis output.

**Composed sub-skills:** `dt-obs-frontends` (AdvancedPerformance, RequestPerformance, RequestTimingAnalysis) + `dt-obs-tracing` (TraceCorrelation)

**Substrate notes:** Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events. Degrades gracefully when metric dimensions are absent (tenant-aggregate scope instead of per-app).

---

### 5. `/dt-k8s-podf` — Pod Debug Forensics

**Syntax:**
```
/dt-k8s-podf NS:"<namespace>" WORKLOAD:"<workload>" [POD:"<pod-name>"] [--notebook] [-clean]
```

**Design intent:** "kubectl describe on steroids" — joins K8s events, container logs, distributed traces, AND applied network policies. Classifies root cause into 10+ classes. The network policy join catches "phantom errors" that other tools miss.

**Composed sub-skills:** `dt-obs-kubernetes` + `dt-obs-logs` + `dt-obs-tracing`. With `--notebook`: also `dt-app-notebooks`.

**Substrate notes:** K8s events surface as `event.kind == "DAVIS_EVENT"` filtered by `k8s.namespace.name` — NO `dt.system.bucket == "default_events"` filter.

---

## Tier 2 — Detailed Reference

### 6. `/dt-cloud-cost` — Cloud Cost Attribution

**Syntax:**
```
/dt-cloud-cost CLOUD:aws|azure|gcp [SCOPE:region:<r>|host:<host-id>] [TAG:<key>] [--list-price] [-clean]
```

**Design intent:** Monthly FinOps attribution report without leaving Dynatrace. Pulls Grail bizevents from the carbon-impact app, sums cost by tag/team/region, computes deltas vs prior period.

**Composed sub-skills:** `dt-obs-aws` / `dt-obs-azure` / `dt-obs-gcp` (cost-optimization references) + `dynatrace.biz.carbon` bizevents

**Substrate notes:** `fetch billing` does NOT exist. Source is `bizevents` with `event.provider == "dynatrace.biz.carbon"`. **Tenant validated:** Azure $152K / AWS $55K over 3 days.

---

### 7. `/dt-rum-journey` — User Journey Funnel Diff

**Syntax:**
```
/dt-rum-journey APP:"<rum-app>" STEPS:"step1>step2>step3" {BASELINE:<iso> COMPARE:<iso>|--vs-deploy DEPLOY-ID} [-clean]
```

**Design intent:** Funnel conversion delta with backend attribution. Identifies the largest drop-off, names the most-frequent NEXT action users took instead of progressing, and correlates to backend errors.

**Composed sub-skills:** `dt-obs-frontends` (UserAction, NavigationPatterns, user-sessions) + `dt-obs-tracing` (TraceCorrelation)

**Substrate notes:** Data source is `user.events`. UserAction matching on `interaction.name` exclusively — `user_action.name` has zero rows on most tenant families.

---

### 8. `/dt-ai-obs` — LLM Application Observability Brief

**Syntax:**
```
/dt-ai-obs APP:"<llm-app>" FROM:<iso> TO:<iso> --framework langgraph|crewai|openai|anthropic [-clean]
```

**Design intent:** Framework-aware analysis of GenAI/LLM apps instrumented with OpenLLMetry (`gen_ai.*` spans). Surfaces per-prompt latency, token-cost outliers, agent-loop detection, and per-feature unit economics.

**Composed sub-skills:** `dt-obs-tracing` (OpenLLMetry spans, request-attributes) + `dt-obs-services`

**Substrate notes:** gen_ai spans confirmed: 58,889 spans in 3d, p50=9ms p95=16ms on validation tenant. Framework-specific attributes: `langgraph.node`, `crewai.task.id`/`crewai.agent`, OTel `gen_ai.system`/`gen_ai.request.model`/`gen_ai.usage.*`.

---

## Shared Substrate

All uber-skills inherit:

| Phase | Behavior |
|---|---|
| 0-auth | `dtctl auth refresh` (silent, never proactive login) |
| 0a | Argument parser; flag table |
| 0b | Minimal sequential bootstrap (anchor resolution, 2-3 queries max) |
| 0c | Reference strategy — each worker reads ONE targeted reference file at start |
| 1 | Parallel workers — ALL dispatched in ONE message with multiple Agent calls |
| 1.5 | Absence-gate — any "not present" claim must be proven by unfiltered confirmation query |
| 2 | Synthesis — hypothesis graph (forensics) or scorecard (risk-scoring) |
| 3 | Davis CoPilot synthesis (where it adds value) |
| 4 | Report generation — MD + PDF via dt-rca PDF rules; Mermaid in MD / ASCII in PDF |
| `-clean` | Sanitization per dt-rca Phase 1.14 |

---

## Tenant-Validated Lessons (2026-05-28)

| # | Lesson |
|---|---|
| 1 | dtctl drops the `{ok, result}` envelope in subprocess — parse with `parsed.get("result", parsed)` |
| 2 | `dt.slo`, `dt.security.events`, `dt.rum.web.events`, `dt.kubernetes.events`, `fetch billing` do NOT exist as Grail data objects |
| 3 | Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events |
| 4 | `dt.system.bucket == "default_events"` returns 0 on this tenant family — drop it |
| 5 | Cost data is in `bizevents` with `event.provider == "dynatrace.biz.carbon"` |
| 6 | Tenant metric dimensions can be absent — skills must degrade gracefully, not fail |
| 7 | UserAction names are on `interaction.name`, not `user_action.name`, on this tenant family |
| 8 | Use `--default-timeframe-start`/`--default-timeframe-end` on `dtctl query` to inject windows — never mutate the DQL string |
