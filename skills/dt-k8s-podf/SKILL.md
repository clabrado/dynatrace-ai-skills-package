---
name: dt-k8s-podf
description: >
  Pod Debug Forensics — "kubectl describe on steroids" for Dynatrace. Given a
  Kubernetes namespace + workload, runs a parallel forensic investigation across
  K8s events, container logs, distributed traces, and applied network policies,
  classifies the root cause (OOMKilled, CrashLoopBackOff, ImagePullBackOff, probe
  failure, denied egress, DNS resolution failure, app exception, scheduling/resource-
  pressure, config drift), and produces either a polished MD+PDF forensic report
  (default) or a deployable Dynatrace Notebook (--notebook).
---

# dt-k8s-podf — Kubernetes Pod Debug Forensics

Agentic, parallel forensic skill for a single Kubernetes workload. Anchors on
`NS:"<namespace>" WORKLOAD:"<workload>" [POD:"<pod-name>"]`, fans out four
parallel workers (events, logs, traces, network), synthesizes a typed root cause
classification, asks Davis CoPilot for a cause-class-specific remediation
runbook, then ships either a Markdown + PDF report (default) or a deployed
Dynatrace Notebook (`--notebook`).

## Usage

```
/dt-k8s-podf NS:"checkout" WORKLOAD:"checkout-api"
/dt-k8s-podf NS:"payments" WORKLOAD:"payment-worker" POD:"payment-worker-7d4f9-xkj2p"
/dt-k8s-podf NS:"frontend" WORKLOAD:"web-ui" --notebook
/dt-k8s-podf NS:"orders" WORKLOAD:"order-svc" -clean
```

## Arguments

| Argument | Required | Description |
|---|---|---|
| `NS:"<namespace>"` | yes | Kubernetes namespace (`k8s.namespace.name`) |
| `WORKLOAD:"<workload>"` | yes | K8s workload name — Deployment/StatefulSet/DaemonSet/Job |
| `POD:"<pod-name>"` | optional | Narrow scope to a single pod |
| `--notebook` | optional | Deploy a Dynatrace Notebook instead of MD+PDF |
| `-clean` | optional | Sanitize all identifying names |
| `appendix=true` | optional | Full Appendix B (Glossary) + C (Capabilities) |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with K8s integration (OneAgent on the cluster)
- dynatrace-for-ai skills: `dt-obs-kubernetes`, `dt-obs-logs`, `dt-obs-tracing`
- `dt-pr-notebooks` skill (only for `--notebook` mode)
- `md-to-pdf` for PDF output

## Sub-Skills Loaded Per Phase

| Worker | Reference files (read ONCE at worker start) |
|---|---|
| W-events | `dt-obs-kubernetes/references/workload-health.md` + `pod-debugging.md` |
| W-logs   | `dt-obs-logs/SKILL.md` |
| W-traces | `dt-obs-tracing/references/failure-detection.md` (+ `http-spans.md`/`rpc-spans.md`/`database-spans.md` by detected protocol) |
| W-network | `dt-obs-kubernetes/references/network-policies.md` + `labels-annotations.md` |

## K8S SCOPING (NON-NEGOTIABLE)

Every events/logs/spans query MUST include:
```dql
| filter k8s.namespace.name == "{NS}"
| filter k8s.workload.name  == "{WORKLOAD}"
[ | filter k8s.pod.name      == "{POD}" ]  -- only if POD provided
```

**Time field discipline:**
- Logs filter on `timestamp` (NOT `start_time`)
- Spans filter on `start_time`
- Events filter on `timestamp`
- **NO `dt.system.bucket == "default_events"` filter** — returns 0 on this tenant family (validated 2026-05-28)

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0b: Workload Bootstrap (Sequential, 3 queries)

**Q1:** Cloud application + owner labels:
```dql
fetch dt.entity.cloud_application, from:now()-7d
| filter entity.name == "{WORKLOAD}" and contains(toString(tags), "{NS}")
| fieldsAdd entity.name, tags, belongs_to[dt.entity.kubernetes_cluster], metadata.labels, metadata.annotations
| limit 1
```

**Q2:** Container discovery:
```dql
fetch spans, from:now()-6h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| summarize span_count=count(), by: { k8s.container.name }
| sort span_count desc | limit 10
```
If no spans → fall back to logs for container discovery.

**Q3:** Cluster name resolution via `smartscapeNodes KUBERNETES_CLUSTER`.

`SCAN_HOURS = 6` (default; widen to 24 if POD is named and no longer exists).

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

W-events, W-logs, W-traces, W-network run concurrently. Skip W-traces if `SPAN_PRESENCE == false` AND no instrumentation expected.

**W-events queries (ALL in one parallel batch):**
1. All workload events (last `SCAN_HOURS`):
```dql
fetch events, from:now()-{SCAN_HOURS}h
| filter k8s.namespace.name == "{NS}" and k8s.workload.name == "{WORKLOAD}"
| fields ts=timestamp, event.type, event.kind, event.name, event.description,
         k8s.pod.name, k8s.container.name, reason
| sort ts desc | limit 200
```

2. Typed event counters (OOMKilled, ImagePullBackOff, CrashLoopBackOff, FailedScheduling, Evicted, liveness_failed, readiness_failed, restarts).
3. Per-pod restart timeline (15-min bins).

**W-logs queries (ALL in one parallel batch):**
1. ERROR/WARN logs with JSON parse (limit 30)
2. Pattern breakdown: exceptions, timeouts, OOM, DNS failures, connection refused, permission denied, panic
3. Log error-rate timeseries (5-min bins)
4. Per-container error distribution

**W-traces queries (ALL in one parallel batch):**
1. Failure reason taxonomy (`dt.failure_detection.results`)
2. Exception types from `span.events`
3. HTTP status / endpoint failure distribution
4. Outbound dependency latency
5. Span schema discovery (drives protocol-specific reference selection)

**W-network queries (ALL in one parallel batch):**
1. Applied NetworkPolicies in namespace
2. Pod-egress denial events (matchesPhrase for NetworkPolicy/policy denied/egress)
3. Outbound destinations attempted (client spans, fail_count)
4. DNS resolution failures ("no such host", "ECONNREFUSED", "EAI_AGAIN")

### Phase 1.5: Absence Gate

For "no events / no logs / no spans / no policies" claims: run unfiltered confirmation (drop WORKLOAD filter, keep NS). If other workloads return data but this one doesn't, that's a genuine workload-scoped absence.

### Phase 2: Cause Classification

Classify into ONE primary class (first match wins):

| Class | Trigger signals |
|---|---|
| **resource-oom** | `oom_killed > 0` OR `pattern_counts.oom > 0` OR exception type contains "OutOfMemory" |
| **resource-cpu-throttle** | Periodic restarts, no app exceptions, elevated outbound latency, no policy denials |
| **image-pull** | `image_pull_backoff > 0` |
| **scheduling** | `failed_scheduling > 0` OR `evicted > 0` |
| **network-denied-egress** | `egress_denied_suspected == true` OR (dns_failures > 0 AND applied_policies non-empty) |
| **network-dns** | `dns_failures > 0` AND `applied_policies` empty (cluster DNS issue) |
| **probe-config** | `liveness_failed + readiness_failed > 0` AND zero app exceptions AND zero failed spans |
| **app-code** | `top_exceptions[0].count > 0` with trace evidence AND no resource/network signals |
| **dependency** | Outbound p95 elevated AND 5xx failures |
| **config-drift** | Recent restart correlates with permission_denied / 403 logs |
| **inconclusive** | None of the above signals fire |

**Confidence scoring:**
- 1.0 — ≥3 corroborating signals across ≥2 phases
- 0.7 — 2 signals in 1 phase
- 0.4 — 1 signal only
- < 0.4 → `inconclusive`

### Phase 3: Davis CoPilot Cause-Class Runbook

Pass cause classification + evidence to Davis (NOT raw findings). Ask for: concrete remediation steps (kubectl/YAML), owner team identification, single verification observable, risk + rollback plan.

### Phase 4a: Default Mode (MD+PDF)

**Sections (in order):**
1. H1 + H2 with cause class
2. Header: Generated / Analyst / Environment / Anchor / Cluster / Owner / Window / Phases
3. Executive Summary (Root Cause Class, Impact Summary, Immediate Action, Restart Timeline gantt)
4. Workload Topology (ASCII box diagram)
5. Event Timeline (K8s events)
6. Log Excerpts (error patterns + sample messages + error-rate trend)
7. Span Errors (failure taxonomy + exception types + failing endpoints + outbound degradation)
8. Network Policy Analysis (applied policies + denial events + destinations + DNS failures)
9. Cross-Phase Evidence Chain (the single authoritative evidence table)
10. Davis CoPilot Synthesis
11. Recommended Actions (P1 + Follow-Up + Verification observable)
12. Appendix A: Investigation Details + worker telemetry
13. Links + *End of Report*

**Filenames:**
```
Normal: PODF_{NS}_{WORKLOAD}_{DATE}.md / .pdf
Clean:  PODF_{NS}_{WORKLOAD}_{DATE}_SANITIZED.md / .pdf
```

### Phase 4b: Notebook Mode (--notebook)

Builds 17-section notebook using `dt-pr-notebooks` Phase 4 authoring pattern. Sections: Executive Summary → K8s Events → Container Logs → Span Errors → Network Policy → Root Cause Classification → Remediation → Post-fix Verification DQL.

All DQL validated before embedding. Verification query uses `from:now()-15m` (live state). Notebook deployed via:
```bash
bash ~/.claude/skills/dt-app-notebooks/scripts/deploy_notebook.sh /tmp/{NS}-{WORKLOAD}-podf.json
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **K8s anchor is canonical scope** — `k8s.namespace.name` + `k8s.workload.name` on every query. Never scope by `dt.entity.service` alone.
2. **Logs on `timestamp`, spans on `start_time`** — wrong time field returns zero rows.
3. **No `dt.system.bucket == "default_events"` filter** — confirmed returns 0 on most tenant families.
4. **Container filter required for multi-container workloads** — prevents istio-proxy access-log noise.
5. **Cause classification is mandatory** — typed `primary_cause` from defined classes. `inconclusive` is the only allowed free-form value.
6. **NetworkPolicy phantom-error check is mandatory** — W-network runs on every investigation.
7. **Verification observable is required** — must define a single observable that returns 0 once the fix is applied.
8. **`--notebook` and MD+PDF are mutually exclusive** — one artifact set per invocation.
9. **Filename = `PODF_*`** — never Problem_* or RCF_*.

## Real-World Use Cases

1. **3am CrashLoopBackOff** — SRE wakes up; skill produces a sourced root cause class in 60s.
2. **Network policy mystery** — "why does service X get connection refused only in prod?" — surfaces the denied egress.
3. **OOM kill recurrence** — three OOM kills this week; skill produces a trend + JVM/runtime memory signal for capacity planning.
4. **Customer onboarding** — new K8s customer hits scheduling pressure; SE runs the skill, hands them a notebook for self-service investigation.
5. **Post-incident retro** — `--notebook` mode produces a re-runnable forensic artifact for the postmortem.

## Substrate Notes (Validated 2026-05-28)

- K8s events confirmed as `event.kind == "DAVIS_EVENT"` filtered by `k8s.namespace.name` — no bucket filter needed
- Tenant has rich event data: `problem-patterns` ns (17,810 events), `unguard` (1,291), `parallel-processing` (355)
- `user.events` path for W-attribution is valid (logs path for containers is also valid)
