# Dynatrace AI Skills Package

A collection of Claude Code skills for proactive, agentic observability on Dynatrace — authored and tenant-validated by Chris LaBrado, Lead Solutions Engineer.

These skills compose Dynatrace telemetry (spans, logs, metrics, events, bizevents, RUM, AppSec, SLOs) with agentic parallel workers and Davis CoPilot synthesis to produce polished MD+PDF artifacts, deployed notebooks, and deployed dashboards — all from a single slash command.

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

## Skill Catalog

### Incident-Anchored Skills

| Skill | Anchor | Output |
|---|---|---|
| [`/dt-rcf`](skills/dt-rcf/SKILL.md) | Problem ID, Service, Time window, Trace ID, or Error pattern | Forensic RCA report (MD+PDF) |
| [`/dt-pr-notebooks`](skills/dt-pr-notebooks/SKILL.md) | Problem ID | Deployed Dynatrace Notebook |
| [`/dt-pr-dashboard`](skills/dt-pr-dashboard/SKILL.md) | Problem ID | Deployed Dynatrace Dashboard |

### Tier 1 — Proactive / Pre-Emptive

| Skill | Anchor | Output | Validated |
|---|---|---|---|
| [`/dt-slo-burn`](skills/dt-slo-burn/SKILL.md) | SLO ID or name | Burn briefing (MD+PDF) | ✅ |
| [`/dt-vuln-blast`](skills/dt-vuln-blast/SKILL.md) | CVE or `library@version` | Blast radius report (MD+PDF) + optional GitHub PR | ✅ |
| [`/dt-deploy-risk`](skills/dt-deploy-risk/SKILL.md) | Deploy event ID or Service+Version | Risk scorecard (MD+PDF) | ✅ |
| [`/dt-dem-vitals`](skills/dt-dem-vitals/SKILL.md) | RUM app + time windows | Web Vitals regression brief (MD+PDF) | ✅ |
| [`/dt-k8s-podf`](skills/dt-k8s-podf/SKILL.md) | Namespace + workload | Pod forensics (MD+PDF or Notebook) | ✅ |

### Tier 2 — Specialized

| Skill | Anchor | Output | Validated |
|---|---|---|---|
| [`/dt-cloud-cost`](skills/dt-cloud-cost/SKILL.md) | Cloud + scope + tag | FinOps attribution report (MD+PDF) | ✅ |
| [`/dt-rum-journey`](skills/dt-rum-journey/SKILL.md) | RUM app + funnel steps + windows | User journey funnel diff (MD+PDF) | ✅ |
| [`/dt-ai-obs`](skills/dt-ai-obs/SKILL.md) | LLM app + window + framework | LLM observability brief (MD+PDF) | ✅ |

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

> **Note:** Skills are `.md` files inside named directories. Claude Code auto-discovers them from `~/.claude/skills/`.

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

# Monthly cloud cost attribution by team tag
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
         ──▶ Phase 0-auth: dtctl auth refresh (silent)
         ──▶ Phase 0b: sequential bootstrap (anchor resolution)
         ──▶ Phase 0c: reference strategy (one reference file per worker)
         ──▶ Phase 1: parallel worker dispatch (ALL in ONE message)
         ──▶ Phase 1.5: absence-gate (no "no data" without proof)
         ──▶ Phase 2: synthesis (hypothesis graph / scorecard)
         ──▶ Phase 3: Davis CoPilot synthesis
         ──▶ Phase 4: MD + PDF report (or notebook/dashboard deploy)
```

**Key design principles:**
- Workers never authenticate — orchestrator-only auth via on-disk token
- Workers read ONE reference file from dynatrace-for-ai at start — DQL stays current
- Absence-gate prevents false "no data" findings
- All skills are reference-driven — no hardcoded DQL

---

## Tenant-Validated Lessons

All 8 Tier 1/2 skills were validated against a live Dynatrace tenant (2026-05-28). Key substrate lessons:

| # | Finding |
|---|---|
| 1 | `fetch dt.slo`, `fetch billing`, `fetch dt.rum.web.events`, `fetch dt.kubernetes.events` — these do NOT exist. Use unprefixed forms (`security.events`, `bizevents`, `events`) or `dtctl get/describe` for settings resources. |
| 2 | Web Vitals are **metrics** (`dt.frontend.web.page.*`), not RUM events |
| 3 | `dt.system.bucket == "default_events"` returns 0 on most tenant families — drop it |
| 4 | Cost data is in `bizevents` with `event.provider == "dynatrace.biz.carbon"` — not a `billing` data object |
| 5 | UserAction names match on `interaction.name`, not `user_action.name` |
| 6 | Use `--default-timeframe-start/--default-timeframe-end` on `dtctl query` to inject windows — never mutate the DQL string |

See [SKILLS_REFERENCE.md](SKILLS_REFERENCE.md) for the full detailed reference.

---

## Author

**Chris LaBrado** — Lead Solutions Engineer, Dynatrace  
Tenant-validated: 2026-05-28  
Substrate authority: `/dt-rcf` SKILL.md

---

## Dependencies

| Dependency | Required | Purpose |
|---|---|---|
| [dtctl](https://github.com/dynatrace-oss/dtctl) | ✅ Required | Dynatrace CLI for DQL queries and resource management |
| [dynatrace-for-ai](https://github.com/Dynatrace/dynatrace-for-ai) | ✅ Required | Reference skill files consumed by worker agents |
| [md-to-pdf](https://github.com/simonhaenisch/md-to-pdf) | ✅ Required for PDF output | Converts markdown reports to PDF |
| GitHub MCP | Optional | Required only for `/dt-vuln-blast --pr/--apply` |
