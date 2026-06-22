# Dynatrace AI Skills Package

A collection of AI agent skills for proactive, agentic observability on Dynatrace — compatible with **Claude Code**, **GitHub Copilot**, **OpenAI Codex**, and any MCP-compatible AI client. Authored and tenant-validated by Chris LaBrado, Lead Solutions Engineer, Dynatrace.

These skills compose Dynatrace telemetry (spans, logs, metrics, events, bizevents, RUM, AppSec, SLOs) with agentic parallel workers and Davis CoPilot synthesis to produce polished Markdown artifacts (PDF optional via `--pdf`), deployed notebooks, and deployed dashboards — all from a single slash command. Each skill follows a reference-driven architecture: workers read one targeted DQL reference file at runtime, so queries stay current without hardcoding.

---

## Skill Catalog

### Incident-Anchored Skills

#### [`/dt-rcf`](skills/dt-rcf/SKILL.md) — Root Cause Forensics
Accepts any forensic anchor — Problem ID, service name, time window, trace ID, or error pattern — and dispatches four parallel workers across spans, logs, services, and Davis problems telemetry. Builds a ranked hypothesis graph with confidence scores (span evidence × log corroboration × runtime signal × recurrence history), synthesizes findings via Davis CoPilot, and produces a complete Markdown forensic report (PDF optional via `--pdf`) with evidence chain. Use when Davis hasn't fired yet, when RCA spans multiple services without a clear anchor, or when you need a defensible evidence chain rather than a point-in-time guess.

#### [`/dt-pr-notebooks`](skills/dt-pr-notebooks/SKILL.md) — Problem to Deployed Notebook
Given a Dynatrace Problem ID, queries live telemetry to identify root cause, builds an ASCII topology diagram of the cascade, and deploys a Dynatrace Notebook with step-by-step runnable DQL queries paired with SRE-readable annotations. Each section explains what to look for, what healthy looks like, and what the current state means for the incident. Use when you want a persistent, re-runnable artifact the owning team can use for self-service investigation or postmortem walkthroughs.

#### [`/dt-pr-dashboard`](skills/dt-pr-dashboard/SKILL.md) — Problem to Deployed Dashboard
Given a Dynatrace Problem ID, diagnoses root cause from live telemetry and deploys a Dynatrace Dashboard optimized for simultaneous operator visibility during an active incident. Tile types are selected by problem category — ERROR gets error rate trend + log volume + HTTP status distribution; SLOWDOWN gets latency percentiles + slow endpoint table; AVAILABILITY gets entity health honeycomb — with a fixed verification row that shows zero when the fix has taken hold. Use when you want a live ops view pinned to the incident window that any operator can open and interpret immediately.

---

### Tier 1 — Proactive / Pre-Emptive

#### [`/dt-slo-burn`](skills/dt-slo-burn/SKILL.md) — Error-Budget Burn Briefing ✅ Validated
Computes multi-window burn rates per the Google SRE Workbook (fast: 1h + 5m @ 14.4×, slow: 6h + 30m @ 6×) against any Custom SLI SLO, identifies the top contributing endpoints and consumers, and surfaces correlated open Davis problems. Optionally projects the budget-exhaustion datetime using Dynatrace predictive analytics, giving a concrete "we'll breach at 14:32 UTC" answer rather than just a rate. Produces a GO / SLOW BURN / FAST BURN / FREEZE verdict with a sourced Markdown briefing (PDF optional via `--pdf`) — use before a release, as a CI deploy gate, during a mid-incident exec brief, or for weekly SRE budget reviews.

#### [`/dt-vuln-blast`](skills/dt-vuln-blast/SKILL.md) — Vulnerability Blast Radius ✅ Validated
Turns a CVE or library coordinate into a prioritized fix list of production services, ranked by real reachability evidence from the span call graph — not just CVSS score — weighted by traffic volume, internet exposure, and error-path involvement. Maps each affected service to its owning team via Kubernetes labels, computes a weighted risk score (reachability 35% / traffic 25% / internet 20% / error-path 20%), and can optionally draft a manifest upgrade patch as a GitHub draft PR. Use for AppSec triage at scale, sprint planning by exploitability, Log4Shell-class supply-chain response, or customer-facing AppSec demos in `-clean` mode.

#### [`/dt-deploy-risk`](skills/dt-deploy-risk/SKILL.md) — Deployment Risk Scorecard ✅ Validated
Compares equivalent pre- and post-deploy windows across RED metrics, new exception types (set-difference, so pre-existing errors don't inflate the score), new log patterns, and downstream dependency health changes. Scores 0–100 via a weighted rubric (error rate 35% / latency 25% / new exceptions 25% / dependency churn 15%) and assigns a GO / HOLD / ROLLBACK verdict with a sourced scorecard suitable for attaching to a change request. Use as a self-service post-deploy sanity check, a CI webhook gate, or a rollback decision audit trail — before the next page fires.

#### [`/dt-dem-vitals`](skills/dt-dem-vitals/SKILL.md) — Web Vitals Regression Brief ✅ Validated
Diffs Core Web Vitals (INP, LCP, CLS, FCP, TTFB) between a baseline and compare window for a Dynatrace RUM application, applying material-delta thresholds to distinguish real regressions from noise. Classifies each regression as frontend-rendered, backend-bound, or network-bound by correlating regressed pages to specific XHR slowdowns and the responsible backend services via distributed trace attribution — ending "is it frontend or backend?" debates with evidence. Use after any deploy, for A/B variant rollout analysis, for Black Friday traffic shift comparisons, or for any customer conversation where a web vitals regression needs to be attributed to an owner team.

#### [`/dt-k8s-podf`](skills/dt-k8s-podf/SKILL.md) — Kubernetes Pod Debug Forensics ✅ Validated
Runs a parallel forensic investigation across K8s events, container logs, distributed traces, and applied NetworkPolicies for a given namespace and workload, then classifies the root cause into one of ten typed classes: OOMKilled, CrashLoopBackOff, ImagePullBackOff, probe failure, denied egress, DNS resolution failure, app exception, scheduling pressure, config drift, or inconclusive. The NetworkPolicy join is what differentiates this from `kubectl describe` — it surfaces "phantom errors" where an app throws connection refused or DNS failures because egress to a dependency is blocked, not because the app itself is broken. Produces either a Markdown forensic report (PDF optional via `--pdf`) with a cause-class-specific Davis CoPilot runbook, or a deployable Dynatrace Notebook for self-service investigation.

---

### Tier 2 — Specialized

#### [`/dt-cloud-cost`](skills/dt-cloud-cost/SKILL.md) — Cloud Cost Attribution (FinOps) ✅ Validated
Pulls AWS, Azure, or GCP cost data from the Dynatrace carbon-impact app bizevents, attributes spend to teams or projects by joining host entities to their Smartscape tag values, and computes period-over-period deltas to surface top movers, new initiatives, and decommissioned workloads. Produces a FinOps attribution report entirely within Dynatrace — no separate FinOps tool required — answering "who spent the most, who changed the most, and what's new this month" in one artifact. Use for monthly chargeback/showback reporting, budget anomaly triage, untagged spend audits (chargeback compliance gaps are surfaced explicitly), or cloud cost demos in `-clean` mode.

#### [`/dt-rum-journey`](skills/dt-rum-journey/SKILL.md) — User Journey Funnel Diff ✅ Validated
Diffs user journey conversion rates step-by-step between two time windows, pinpoints the biggest drop-off, identifies the most-frequent next action leakers took instead of progressing, and correlates the drop-off to backend errors using a differential rate comparison — leaker failure rate vs. progressor failure rate — to distinguish "backend errors caused the abandonment" from "backend errors that happen to everyone." Use for cart abandonment investigations, onboarding funnel regressions, deploy-induced UX breaks (via `--vs-deploy`), or A/B variant analysis where conversion impact needs to be measured with backend attribution.

#### [`/dt-ai-obs`](skills/dt-ai-obs/SKILL.md) — LLM Application Observability Brief ✅ Validated
Framework-aware observability brief for GenAI and LLM apps instrumented with OpenLLMetry, covering per-model latency (p50/p95/p99), token cost outliers with embedded pricing tables, agent-loop detection (repeated identical prompts within a session across both content-hash and step-frequency dimensions), and per-feature unit economics expressed as cost-per-request. Supports LangGraph, CrewAI, OpenAI SDK, and Anthropic SDK span attribute sets; unknown models and stale pricing are flagged rather than silently underreported. Use when an LLM cost bill spikes, an agent is running slow or stuck in a reflexion loop, you need to compare model ROI across frameworks, or you're sizing production capacity for a new LLM feature.

---


### Foundations & Building Blocks (added 2026-06-22)

Lower-level query and builder skills the analysis suite composes on — observability data access, dashboard/notebook builders, DQL guidance, and cloud governance.

- [`/dt-obs-hosts`](skills/dt-obs-hosts/SKILL.md) — Host and process metrics including CPU, memory, disk, network, containers, and process-level telemetry.
- [`/dt-obs-services`](skills/dt-obs-services/SKILL.md) — Service performance monitoring with RED metrics (Rate, Errors, Duration) and runtime-specific telemetry for Java, .NET, Node.js, Python, PHP, and Go.
- [`/dt-obs-logs`](skills/dt-obs-logs/SKILL.md) — Log querying, filtering, pattern analysis, and error rate calculation.
- [`/dt-obs-tracing`](skills/dt-obs-tracing/SKILL.md) — Distributed traces, spans, service dependencies, and request flow analysis.
- [`/dt-obs-problems`](skills/dt-obs-problems/SKILL.md) — DAVIS problem analysis including root cause identification, impact assessment, and correlation with other telemetry.
- [`/dt-obs-kubernetes`](skills/dt-obs-kubernetes/SKILL.md) — Kubernetes cluster, pod, node, and workload monitoring.
- [`/dt-obs-aws`](skills/dt-obs-aws/SKILL.md) — AWS cloud resource monitoring including EC2, RDS, Lambda, ECS/EKS, VPC networking, load balancers, S3, DynamoDB, SQS/SNS, and cost optimization.
- [`/dt-obs-azure`](skills/dt-obs-azure/SKILL.md) — Azure cloud resources including VMs, VMSS, SQL Database, Storage, AKS, App Service, Functions, VNet networking, load balancers, Event Hubs, Container Apps,…
- [`/dt-obs-gcp`](skills/dt-obs-gcp/SKILL.md) — GCP cloud resources including Compute Engine, GKE, Cloud Run, Pub/Sub, VPC networking, DNS, IAM, Secret Manager, and monitoring.
- [`/dt-obs-frontends`](skills/dt-obs-frontends/SKILL.md) — Real User Monitoring (RUM), Web Vitals, user sessions, mobile crashes, page performance, user interactions, and frontend errors.
- [`/dt-obs-log-parser`](skills/dt-obs-log-parser/SKILL.md) — Log pattern detection, DQL parse statement generation, and OpenPipeline processing rule deployment.
- [`/dt-obs-predictive-analytics`](skills/dt-obs-predictive-analytics/SKILL.md) — Predictive analytics for Dynatrace — time series forecasting with the timeseries-forecast tool, capacity saturation planning, trend and anomaly detection a…
- [`/dt-dql-essentials`](skills/dt-dql-essentials/SKILL.md) — Core DQL syntax rules, common pitfalls, and query patterns.
- [`/dt-app-dashboards`](skills/dt-app-dashboards/SKILL.md) — Work with Dynatrace dashboards - create, modify, query, and analyze dashboard JSON including tiles, layouts, DQL queries, variables, and visualizations.
- [`/dt-app-notebooks`](skills/dt-app-notebooks/SKILL.md) — Work with Dynatrace notebooks - create, modify, query, and analyze notebook JSON.
- [`/dt-migration`](skills/dt-migration/SKILL.md) — Migrate Dynatrace classic and Gen2 entity-based DQL to Smartscape equivalents.
- [`/dt-cloud-compliance`](skills/dt-cloud-compliance/SKILL.md) — Verify Dynatrace internal cloud resource compliance after any cloud work.

## ⚠️ 2026-06-03 UPDATE — change notes

All 8 Tier 1/2 skills were re-validated end-to-end on 2026-06-03 and corrected. The headline changes:

- **Output is Markdown-only by default.** Every artifact ships as Markdown; PDF is opt-in via `--pdf` and non-fatal — a missing `md-to-pdf` engine logs one line and the `.md` still ships.
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

## ⚠️ Prerequisites

This skills package has **two hard dependencies** that must be installed and working before these skills will function:

### 1. `dtctl` CLI or Dynatrace MCP

Every skill queries Dynatrace data via `dtctl`, the kubectl-style CLI for Dynatrace.

**Install dtctl:**
```bash
# Homebrew (macOS/Linux)
brew install dynatrace-oss/tap/dtctl

# Direct install
curl -fsSL https://raw.githubusercontent.com/dynatrace-oss/dtctl/main/install.sh | bash
```

**Configure dtctl** with your Dynatrace tenant and OAuth token:
```bash
dtctl config create-context my-tenant \
  --environment https://YOUR_TENANT.apps.dynatrace.com \
  --token YOUR_OAUTH_TOKEN

dtctl config use-context my-tenant
dtctl auth whoami
```

> **Token storage:** All skills use `DTCTL_TOKEN_STORAGE=file` (on-disk token, never Keychain).  
> Set this in your Claude Code `settings.json` env block:
> ```json
> { "env": { "DTCTL_TOKEN_STORAGE": "file" } }
> ```

### 2. dynatrace-for-ai Skills

These skills consume reference files from the [dynatrace-for-ai](https://github.com/Dynatrace/dynatrace-for-ai) skills package as their DQL authority. The worker agents read those reference files at runtime.

**Install dynatrace-for-ai skills:**
```bash
# Clone the repo
git clone https://github.com/Dynatrace/dynatrace-for-ai.git

# Install skills to Claude Code
cp -r dynatrace-for-ai/skills/* ~/.claude/skills/
# or install via Claude Code's plugin system
```

Skills consumed by this package:
- `dt-obs-problems` — Davis problem patterns, trending, impact analysis
- `dt-obs-services` — RED metrics, service health, runtime metrics
- `dt-obs-tracing` — Span failure detection, trace correlation, sampling
- `dt-obs-logs` — Log search, filtering, pattern analysis
- `dt-obs-frontends` — RUM: Web Vitals, UserActions, sessions, navigation
- `dt-obs-kubernetes` — Pod debugging, workload health, network policies, labels
- `dt-obs-aws` / `dt-obs-azure` / `dt-obs-gcp` — Cloud resource ownership and cost optimization
- `dt-obs-predictive-analytics` — Timeseries forecast (used by `/dt-slo-burn --forecast`)
- `dt-app-notebooks` — Dynatrace Notebook JSON authoring and deploy
- `dt-app-dashboards` — Dynatrace Dashboard JSON authoring and deploy
- `dt-dql-essentials` — DQL syntax, smartscapeNodes, getNodeName signatures

---

## Installation

```bash
# 1. Clone this repo
git clone https://github.com/clabrado/dynatrace-ai-skills-package.git

# 2. Copy skills to Claude Code skills directory
cp -r dynatrace-ai-skills-package/skills/* ~/.claude/skills/

# 3. Verify skills are discoverable (restart Claude Code session if needed)
ls ~/.claude/skills/ | grep dt-
```

> **Note:** Skills are `.md` files inside named directories. Claude Code and other MCP-compatible AI clients auto-discover them from `~/.claude/skills/` or the equivalent configured skills directory.

---

## Quick Start

```bash
# Check SLO burn rate before a release
/dt-slo-burn SLO:"checkout-availability"

# Triage a CVE across your production services
/dt-vuln-blast CVE-2024-12345

# Post-deploy risk scorecard
/dt-deploy-risk DEPLOY:event-abc123

# Web Vitals regression after a deploy
/dt-dem-vitals APP:"www.acme.com" --vs-deploy DEPLOY-12345

# Pod crash forensics
/dt-k8s-podf NS:"payments" WORKLOAD:"checkout-api"

# Monthly cloud cost attribution (TAG is optional — bare CLOUD: runs in dimensional mode)
/dt-cloud-cost CLOUD:azure
/dt-cloud-cost CLOUD:aws TAG:team

# User funnel drop-off analysis
/dt-rum-journey APP:"checkout-web" STEPS:"home>cart>checkout>confirm" \
  BASELINE:"2026-05-20T00:00Z..2026-05-21T00:00Z" \
  COMPARE:"2026-05-27T00:00Z..2026-05-28T00:00Z"

# LLM cost and loop analysis
/dt-ai-obs APP:"chat-assistant" FROM:2026-05-25T00:00Z TO:2026-05-28T00:00Z --framework langgraph

# Full forensic RCA from any anchor
/dt-rcf SERVICE:"checkout-api"
/dt-rcf P-26012345
```

All skills support `-clean` for sanitized customer-shareable output.

---

## Architecture

All skills share the same phased execution substrate inherited from `/dt-rcf`:

```
ANCHOR ──▶ Phase 0a: parse args
         ──▶ Phase 0-auth: dtctl auth refresh (orchestrator-only, interactive)
         ──▶ Phase 0b: sequential bootstrap (anchor resolution)
         ──▶ Phase 0b.0: substrate probe (discover live event.kind / fields / vocabulary)
         ──▶ Phase 0c: reference strategy (one reference file per worker)
         ──▶ Phase 1: parallel worker dispatch (ALL in ONE message)
         ──▶ Phase 1.5: absence-gate (no "no data" without proof)
         ──▶ Phase 2: synthesis (hypothesis graph / scorecard)
         ──▶ Phase 3: Davis CoPilot synthesis
         ──▶ Phase 4: Markdown report (PDF optional via --pdf; or notebook/dashboard deploy)
```

**Key design principles:**
- Workers never authenticate — auth is orchestrator-only, interactive, and never races concurrent logins (workers never run `dtctl auth login`/`refresh`)
- Probe-then-query — Phase 0b.0 discovers the live `event.kind` / field / vocabulary at runtime, so skills self-heal against substrate drift instead of hardcoding a snapshot
- Workers read ONE reference file from dynatrace-for-ai at start — DQL stays current without drift
- Absence-gate prevents false "no data" findings — every absence claim requires unfiltered proof; a pre-dispatch reality gate refuses to fan out workers if Phase 0b couldn't run a successful query, and every worker treats "no data" as a `<gap>`, never a fabricated number
- All skills are reference-driven — no hardcoded DQL in the skill files themselves

---

## Tenant-Validated Lessons

All 8 Tier 1/2 skills were validated against a live Dynatrace tenant (2026-05-28) and re-validated end-to-end + corrected on 2026-06-03 (see the ⚠️ banner above). Key substrate lessons:

| # | Finding |
|---|---|
| 1 | `fetch dt.slo`, `fetch billing`, `fetch dt.rum.web.events`, `fetch dt.kubernetes.events` — these do NOT exist. Use unprefixed forms (`security.events`, `bizevents`, `events`) or `dtctl get/describe` for settings resources. |
| 2 | Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events; windowed `timeseries percentile(...)` requires an explicit `interval:` or it mis-buckets over short windows |
| 3 | `dt.system.bucket == "default_events"` returns 0 on most tenant families — drop it; K8s events surface as `event.kind == "DAVIS_EVENT"` |
| 4 | Cost data is in `bizevents` with `event.provider == "dynatrace.biz.carbon"` — not a `billing` data object; default to the populated `cost.list.price` feed and attribute by native dimensions (`dt.entity.host` is NULL on cost events) |
| 5 | UserAction names match on `interaction.name`, not `user_action.name` |
| 6 | Use `--default-timeframe-start/--default-timeframe-end` on `dtctl query` to inject windows — never mutate the DQL string |
| 7 | **(2026-06-03)** AppSec: CVEs live in the `vulnerability.references.cve` array (match `in(CVE, …)`), `event.kind == "SECURITY_EVENT"`, affected entities in `affected_entity.id` (`PROCESS_GROUP-*`) |
| 8 | **(2026-06-03)** Deploys are `SDLC_EVENT` / `task.deployment.finished` (CUSTOM_DEPLOYMENT is gone; legacy arrives under DAVIS_EVENT); SDLC_EVENT has no service link → bridge `component.name` → SERVICE via spans |
| 9 | **(2026-06-03)** gen_ai `APP` can be a SERVICE or a `gen_ai.request.model` value — resolve both and bind whichever matches; price self-hosted / unknown models at $0 (never collapse the total to $0 and abort) |
| 10 | **(2026-06-03)** A Davis-problem membership filter on a dotted Smartscape field MUST use the function form `in("id", \`field\`)` — the infix form parse-errors |
| 11 | **(2026-06-03)** Never put a bare `$<digit>` in a skill — the harness expands `$0`/`$1`/… as positional-arg variables and corrupts the dollar literal (write `USD 0` instead); and workers must NEVER run `dtctl auth login`/`refresh` (racing non-interactive logins corrupts the token store) |

See [SKILLS_REFERENCE.md](SKILLS_REFERENCE.md) for the full detailed reference with design rationale, real-world use cases, and substrate notes per skill.

---

## Author

**Chris LaBrado** — Lead Solutions Engineer, Dynatrace  
Tenant-validated: 2026-05-28; re-validated end-to-end + corrected: 2026-06-03  
Substrate authority: `/dt-rcf` SKILL.md

---

## Dependencies

| Dependency | Required | Purpose |
|---|---|---|
| [dtctl](https://github.com/dynatrace-oss/dtctl) | ✅ Required | Dynatrace CLI for DQL queries and resource management |
| [dynatrace-for-ai](https://github.com/Dynatrace/dynatrace-for-ai) | ✅ Required | Reference skill files consumed by worker agents |
| [md-to-pdf](https://github.com/simonhaenisch/md-to-pdf) | Optional — only for `--pdf` output (default is Markdown-only) | Converts markdown reports to PDF; if absent, the skill logs one line and ships the `.md` |
| GitHub MCP | Optional | Required only for `/dt-vuln-blast --pr/--apply` |
