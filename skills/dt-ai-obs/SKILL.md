---
name: dt-ai-obs
description: >
  LLM Application Observability Brief — agentic, framework-aware analysis of
  GenAI / LLM apps instrumented with OpenLLMetry (OpenTelemetry semantic
  conventions for LLM spans). Accepts an APP anchor plus time window and a
  framework hint (langgraph, crewai, openai, anthropic). Dispatches four
  parallel workers — latency per prompt/model, token cost outliers, agent-loop
  detection across sessions, and per-feature unit economics ($/feature) — then
  synthesizes via Davis CoPilot. Produces a shareable Markdown + PDF brief.
---

# dt-ai-obs — LLM Application Observability Brief

Agentic LLM observability skill. Reads OpenLLMetry / OpenTelemetry GenAI span
attributes from Dynatrace spans, dispatches four parallel workers, and produces
a Tier-2 brief (Markdown + PDF) covering latency, token cost, agent loops, and
per-feature unit economics.

## Usage

```
/dt-ai-obs APP:"checkout-copilot" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework langgraph
/dt-ai-obs APP:"support-agent" FROM:"2026-05-21T12:00Z" TO:"2026-05-28T12:00Z" --framework crewai
/dt-ai-obs APP:"summarizer-svc" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework openai
/dt-ai-obs APP:"rag-pipeline" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework anthropic -clean
```

## Arguments

| Arg / flag | Description |
|---|---|
| `APP:"<service-or-app-id>"` | Required. Smartscape SERVICE name or service short-id. |
| `FROM:"<iso>"` and `TO:"<iso>"` | Required. ISO-8601 with timezone. |
| `--framework langgraph\|crewai\|openai\|anthropic` | Required. Drives which framework-specific span attributes the workers query. |
| `-clean` | Sanitize all identifying names. Required for any artifact leaving the laptop. |
| `pdf=false` | Skip PDF generation; emit Markdown only. |
| `appendix=true` | Include full Appendix B (Glossary) + C (Pricing Table). |

## Prerequisites

- `dtctl` CLI installed and authenticated (`DTCTL_TOKEN_STORAGE=file`)
- Dynatrace tenant with LLM app instrumented via OpenLLMetry (`gen_ai.*` span attributes)
- dynatrace-for-ai skills: `dt-obs-tracing`, `dt-obs-services`
- `md-to-pdf` for PDF output

## Framework Span Attribute Reference

**Common to all frameworks (OTel GenAI semantic conventions):**

| Attribute | Purpose |
|---|---|
| `gen_ai.system` | `openai`, `anthropic`, `bedrock`, etc. |
| `gen_ai.request.model` | Model identifier (e.g. `gpt-4-turbo`) |
| `gen_ai.usage.input_tokens` | Prompt token count |
| `gen_ai.usage.output_tokens` | Completion token count |
| `gen_ai.request.temperature` | Sampling temperature |

**Framework-specific:**
- `langgraph`: `langgraph.node`, `langgraph.checkpoint`
- `crewai`: `crewai.task.id`, `crewai.agent`, `crewai.role`
- `openai`/`anthropic` (direct SDK): use only `gen_ai.*` attributes

## Sub-Skills Loaded Per Worker

| Worker | Reference files (read ONCE at worker start) |
|---|---|
| W-latency | `dt-obs-tracing/references/performance-analysis.md` + `request-attributes.md` |
| W-cost | `dt-obs-tracing/references/request-attributes.md` |
| W-loops | `dt-obs-tracing/references/request-attributes.md` |
| W-econ | `dt-obs-tracing/references/request-attributes.md` + `dt-obs-services/references/service-metrics.md` |

## PRICING TABLE (baseline — validate before customer-facing artifacts)

| Model | Input ($/1M) | Output ($/1M) | As of |
|---|---|---|---|
| gpt-4 | 30.00 | 60.00 | est. 2026 |
| gpt-4-turbo | 10.00 | 30.00 | est. 2026 |
| gpt-3.5-turbo | 0.50 | 1.50 | est. 2026 |
| claude-opus-4 | 15.00 | 75.00 | est. 2026 |
| claude-sonnet-4 | 3.00 | 15.00 | est. 2026 |
| claude-haiku-4 | 0.25 | 1.25 | est. 2026 |

**Embed pricing table in worker prompts** (W-cost, W-econ). Workers do NOT read it from disk.
**VALIDATE before customer-shareable artifacts** — refresh from provider pricing pages.

## EXECUTION PROTOCOL

### Phase 0-auth: Pre-warm dtctl session

**FIRST action:** `dtctl auth refresh`. Workers never authenticate.

### Phase 0b: Context Bootstrap (Sequential)

**Step 0b.1:** Resolve APP service entity:
```dql
smartscapeNodes SERVICE
| filter contains(name, "{APP_NAME}")
| fields id, name, properties | limit 5
```

**Step 0b.2:** Validate window has spans:
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| summarize span_count = count()
```
If `span_count == 0` → abort (most likely wrong APP name or wrong tenant).

**Step 0b.3:** Detect framework + sample span schema:
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [...]
| filter isNotNull(gen_ai.request.model) or isNotNull(gen_ai.system)
| limit 5
| fields name, gen_ai.system, gen_ai.request.model, langgraph.node, crewai.task.id, attributes
```
If no `gen_ai.*` attributes → abort with "service is not LLM-instrumented".

### Phase 1: Parallel Worker Dispatch (ALL in ONE message)

**ENTITY MODEL — scope spans with:**
```dql
filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
```

**W-latency:** Per-model p50/p95/p99/max latency. Per-step latency by `langgraph.node` / `crewai.task.id` / `span.name`. Latency trend over window.

**W-cost:** Token totals + estimated cost per model (worker computes cost from embedded pricing table). Top-25 most expensive single calls. Cost trend.

**W-loops:** Repeated identical prompts within sessions:
```dql
fetch spans, from:toTimestamp("{from}"), to:toTimestamp("{to}")
| filter ...
| filter isNotNull(gen_ai.prompt.0.content)
| fieldsAdd session_key = coalesce(request_attribute.session.id, request_attribute.conversation.id, toString(trace.id))
| fieldsAdd prompt_hash = hash_md5(gen_ai.prompt.0.content)
| summarize repeats = count(), first_seen = takeMin(start_time), last_seen = takeMax(start_time),
            by: { session_key, prompt_hash, gen_ai.request.model }
| filter repeats >= 3
| fieldsAdd window_minutes = (last_seen - first_seen) / 60000000000
| filter window_minutes <= 5
```

**W-econ:** Token + call volume by feature key (`feature.name`, `endpoint.name`, or `span.name` fallback). Compute `cost_per_request_usd = total_cost_usd / requests`.

### Phase 1.5: Absence Gate

For "no LLM spans / no feature.name" claims: require worker proof + run one unfiltered confirmation.

### Phase 2: Synthesis

Rank findings by severity:
- CRITICAL: p95 > 30s on user-facing step; top-cost feature > 60% of total spend
- HIGH: loop_count > 0; top-N expensive span > $1.00 each
- MEDIUM: high-volume sessions > 50 LLM calls; per-step p95 > 2× per-model p95
- LOW: everything else worth noting

### Phase 3: Davis CoPilot Synthesis

Pass ranked findings (NOT raw PhaseResults) to Davis. Ask for: cheaper-model substitution opportunities, prompt-caching opportunities, loop-break strategies, batching/RAG/output-cap savings.

### Phase 4: Report Generation

**Sections (in order):**
1. H1 + H2 subtitle with APP + FRAMEWORK
2. Header: Generated / Analyst / Environment / Window / Framework / Workers
3. Executive Summary (Headline, Window Totals, Top Concerns, Recommended Immediate Action)
4. Latency Analysis (per-model + per-step + trend)
5. Cost Top-N (per-model breakdown + top-25 expensive calls + trend)
6. Loop Detections (repeated-prompt loops + high-volume sessions + step-loop signatures)
7. Per-Feature Economics (ranked by total cost AND by cost-per-request)
8. Recommended Actions (model substitution + prompt caching + loop-break guardrails)
9. Davis Synthesis
10. Appendix A: Investigation Details + pricing table as-of date + worker telemetry
11. Links + *End of Brief*

**Filenames:**
```
Normal: AIOBS_{APP_SLUG}_{DATE}.md / .pdf
Clean:  AIOBS_{APP_SLUG}_{DATE}_SANITIZED.md / .pdf
```

## QUALITY RULES (NON-NEGOTIABLE)

1. **Pricing table embedded in worker prompts** — W-cost and W-econ carry it verbatim; workers do NOT read from disk.
2. **Unknown-model gap is mandatory** — if W-cost encounters a model not in the pricing table, record `unknown_pricing: <model>` in gaps and exclude from total. Never silently treat as $0.
3. **Framework hint is authoritative** — `--framework` flag drives which attributes workers query; `gen_ai.system` disagreement is logged but doesn't override.
4. **Loop detection has TWO axes** — content-hash (identical prompt cycles) AND step-frequency (same node firing N times). Never ship a brief on only one.
5. **Cost-per-request is required** — W-econ must compute it when FEATURE_KEY resolves.
6. **CLEAN_MODE sanitizes session/trace IDs** — these identify customers when combined with timestamps.
7. **Pricing table `as_of` date in report** — if >30 days since refresh, mark brief as DRAFT.
8. **Filename = `AIOBS_*`** — never RCF_* or Problem_*.

## Real-World Use Cases

1. **LLM cost spike investigation** — bill jumped 3×; top-cost feature + outlier prompts identified in one shot.
2. **Agent loop debugging** — agent runs are slow; loop detector surfaces repeated identical prompts in a session.
3. **Model selection ROI** — comparing gpt-4 vs gpt-4-turbo on same workload; quantifies latency/cost delta.
4. **Pre-launch capacity sizing** — pull p95 latency + token volume to forecast production cost at 10× scale.
5. **Customer AI Observability demo** — `-clean` mode produces a shareable brief showcasing Dynatrace's `gen_ai.*` span support.

## Substrate Notes (Validated 2026-05-28)

- gen_ai spans confirmed: 58,889 spans in 3d, Azure/genai-demo, **p50=9ms p95=16ms**
- Framework-specific attributes: `langgraph.node`, `crewai.task.id`/`crewai.agent`
- OTel attributes: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`
- `gen_ai.*` are OpenTelemetry semantic conventions, not Dynatrace-proprietary — spelling matters
