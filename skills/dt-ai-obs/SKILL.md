---
name: dt-ai-obs
description: >-
  LLM Application Observability Brief — agentic, framework-aware analysis of
  GenAI / LLM apps instrumented with OpenLLMetry (OpenTelemetry semantic
  conventions for LLM spans). Accepts an APP anchor plus time window and a
  framework hint (langgraph, crewai, openai, anthropic). Dispatches four
  parallel workers — latency per prompt/model, token cost outliers, agent-loop
  detection across sessions, and per-feature unit economics ($/feature) — then
  synthesizes via Davis CoPilot. Surfaces gen_ai.request.model, gen_ai.usage.*
  tokens, langgraph.node / crewai.task.id / crewai.agent attributes, repeated
  prompts within a conversation, and the top-cost features. Produces a
  shareable Markdown brief (PDF optional via `--pdf`) with executive
  summary, latency / cost tables, loop detections, per-feature economics,
  and remediation guidance.
  Use when an SE or customer wants a Tier-2 read on an instrumented LLM app:
  what's slow, what's expensive, what's looping, and where the money goes.
---

# dt-ai-obs — LLM Application Observability Brief

Agentic LLM observability skill. Reads OpenLLMetry / OpenTelemetry GenAI span
attributes from Dynatrace spans, dispatches four parallel workers, and produces
a Tier-2 brief (Markdown; PDF optional via `--pdf`) covering latency, token cost, agent loops, and
per-feature unit economics.

This skill is the LLM-application sibling of `/dt-rcf`. It reuses the
substrate: orchestrator-only auth, reference-driven workers, single-message
parallel dispatch, absence-gate proof requirement, dt-rca sanitization + PDF
rules. Differences are framework-specific span attributes and a four-worker
matrix (W-latency, W-cost, W-loops, W-econ) instead of the dt-rcf P/T/L/S set.

---

## Usage

```
/dt-ai-obs APP:"checkout-copilot" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework langgraph
/dt-ai-obs APP:"support-agent" FROM:"2026-05-21T12:00Z" TO:"2026-05-28T12:00Z" --framework crewai
/dt-ai-obs APP:"summarizer-svc" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework openai
/dt-ai-obs APP:"doc-chat" FROM:"2026-05-25T00:00Z" TO:"2026-05-28T00:00Z" --framework anthropic
/dt-ai-obs APP:"rag-pipeline" FROM:"2026-05-27T00:00Z" TO:"2026-05-28T00:00Z" --framework langgraph -clean
/dt-ai-obs APP:"ai-travel-advisor-agent-test" FROM:"2026-05-31T00:00Z" TO:"2026-06-03T00:00Z" --framework openai --pdf
```

> APP may be a Smartscape **SERVICE name** OR a **`gen_ai.request.model`
> value** (e.g. `genai-demo`). Phase 0b resolves both and binds whichever
> matches — see Phase 0b.0.

## Arguments

- `APP:"<service-or-model-value>"` — required. Either a Smartscape SERVICE
  name / short-id, OR a `gen_ai.request.model` VALUE (e.g. `genai-demo`).
  Phase 0b.0 resolves BOTH and binds whichever matches (`APP_BIND_MODE` ∈
  {`service`, `model`}).
- `FROM:"<iso>"` and `TO:"<iso>"` — required. ISO-8601 with timezone.
- `--framework langgraph|crewai|openai|anthropic` — required. Drives which
  framework-specific span attributes the workers query. `openai` and
  `anthropic` cover apps calling those SDKs directly (no agent framework).
- `-clean` — sanitize all identifying names (same rules as `/dt-rca` Phase
  1.14). REQUIRED for any artifact leaving the laptop.
- `--pdf` — ALSO render a PDF (default OFF — Markdown is the canonical
  deliverable). Legacy `pdf=true` is accepted as an alias; `pdf=false` is a
  no-op (MD-only is already the default). PDF is best-effort and never
  blocks a run.
- `appendix=true` — include Appendix B (Glossary) + C (Pricing Table) in full;
  default OFF (one-line pointers instead).

> **Speed note:** run at **medium effort** (not high). The workers do the
> heavy lifting; orchestrator effort doesn't change query quality and only
> multiplies latency.

---

## Sub-Skills Loaded Per Worker

dt-ai-obs carries NO hardcoded LLM DQL. Each worker reads ONE targeted
reference file at start, once per subagent, derives queries from it, applies
the dt-ai-obs scoping rules (Smartscape-native APP filter + time window +
framework attribute set), and reports back.

| Worker     | Reference file (read ONCE at worker start) |
|------------|--------------------------------------------|
| W-latency  | `~/.agents/skills/dt-obs-tracing/references/performance-analysis.md` + `request-attributes.md` |
| W-cost     | `~/.agents/skills/dt-obs-tracing/references/request-attributes.md` |
| W-loops    | `~/.agents/skills/dt-obs-tracing/references/request-attributes.md` |
| W-econ     | `~/.agents/skills/dt-obs-tracing/references/request-attributes.md` + `~/.agents/skills/dt-obs-services/references/service-metrics.md` |

The orchestrator carries no inline DQL except Phase 0b bootstrap and Phase 1.5
Absence-Gate confirmation queries.

---

## Framework Span Attribute Reference

OpenLLMetry follows the OpenTelemetry GenAI semantic conventions. The
following attributes appear on LLM-call spans. Workers query against these
exact names; mark as VALIDATE on first production run (semantic conventions
are still evolving — confirm against current `gen_ai.*` spec).

### Common to all frameworks (OTel GenAI semantic conventions)

| Attribute                       | Purpose                                    |
|---------------------------------|--------------------------------------------|
| `gen_ai.system`                 | `openai`, `anthropic`, `bedrock`, etc.     |
| `gen_ai.request.model`          | Model identifier (e.g. `gpt-4-turbo`)      |
| `gen_ai.response.model`         | Actual responding model (may differ)       |
| `gen_ai.response.id`            | Provider response ID                       |
| `gen_ai.usage.input_tokens`     | Prompt token count                         |
| `gen_ai.usage.output_tokens`    | Completion token count                     |
| `gen_ai.request.temperature`    | Sampling temperature                       |
| `gen_ai.request.max_tokens`     | Max-tokens parameter                       |
| `gen_ai.prompt.<n>.role`        | Per-message role (system / user / assistant) |
| `gen_ai.prompt.<n>.content`     | Per-message content (often hashed in prod) |
| `gen_ai.completion.<n>.content` | Response content                           |

### langgraph

| Attribute                | Purpose                                  |
|--------------------------|------------------------------------------|
| `langgraph.node`         | Graph node name (which step ran)         |
| `langgraph.checkpoint`   | Checkpoint identifier (state versioning) |

### crewai

| Attribute             | Purpose                              |
|-----------------------|--------------------------------------|
| `crewai.task.id`      | Task identifier within a crew run    |
| `crewai.agent`        | Agent name (e.g. `researcher`)       |
| `crewai.role`         | Agent role description               |

### openai / anthropic (direct SDK use)

Use only the `gen_ai.*` set above; `gen_ai.system` is `"openai"` or
`"anthropic"`. Models commonly seen: `gpt-4`, `gpt-4-turbo`, `gpt-3.5-turbo`,
`claude-opus-4`, `claude-sonnet-4`, `claude-haiku-4`.

### Custom request attributes

Apps that wrap LLM calls inside feature handlers often add a custom request
attribute via OneAgent or the OTel SDK — typically `feature.name`, `user.id`,
`tenant.id`, `session.id`, `conversation.id`. These appear as
`request_attribute.<name>` on the request root span (per
`request-attributes.md`). The orchestrator's Phase 0b sample-span query
discovers which custom attributes exist; workers then key off them.

---

## Pricing Table (baseline-estimate — confirm before publishing)

W-cost computes per-span cost as
`input_tokens * input_price + output_tokens * output_price`. The following
baseline prices (USD per 1M tokens) are EMBEDDED for offline computation —
treat them as estimates, refresh from each provider's pricing page before
shipping any customer-facing brief.

The table is split into **priced** (commercial-API) models and a
**self-hosted / unpriced** class. Matching is **prefix/contains, not exact**
(strip provider prefixes like `bedrock/`, version suffixes like `@001`,
`:22b`, `-002`, and date stamps) so `claude-3-5-sonnet-20241022`,
`anthropic.claude-3-sonnet`, and `claude-sonnet-4` all resolve.

#### Priced — commercial APIs (USD per 1M tokens)

| Model (match prefix)        | Input ($/1M) | Output ($/1M) | As of      |
|-----------------------------|--------------|---------------|------------|
| gpt-4o                      |  2.50        | 10.00         | est. 2026  |
| gpt-4-turbo                 | 10.00        | 30.00         | est. 2026  |
| gpt-4                       | 30.00        | 60.00         | est. 2026  |
| gpt-3.5-turbo / gpt-35-turbo |  0.50       |  1.50         | est. 2026  |
| text-embedding-3-large      |  0.13        |  0.00         | est. 2026  |
| text-embedding-3-small      |  0.02        |  0.00         | est. 2026  |
| text-embedding-ada-002      |  0.10        |  0.00         | est. 2026  |
| claude-opus-4 / claude-3-opus       | 15.00 | 75.00  | est. 2026  |
| claude-sonnet-4 / claude-3-5-sonnet |  3.00 | 15.00  | est. 2026  |
| claude-haiku-4 / claude-3-haiku     |  0.25 |  1.25  | est. 2026  |
| claude-2 / claude-2.1               |  8.00 | 24.00  | est. 2026  |
| **Amazon Bedrock** (strip `amazon.` prefix) |     |        |            |
| titan-embed-text            |  0.10        |  0.00         | est. 2026  |
| titan-text-express          |  0.20        |  0.60         | est. 2026  |
| titan-text-lite             |  0.15        |  0.20         | est. 2026  |
| titan-text-premier          |  0.50        |  1.50         | est. 2026  |
| **Google Vertex / Gemini**  |              |               |            |
| gemini-2.5-pro              |  1.25        |  5.00         | est. 2026  |
| gemini-2.0-flash / 1.5-flash |  0.075      |  0.30         | est. 2026  |
| textembedding-gecko         |  0.025       |  0.00         | est. 2026  |

#### Self-hosted / compute-billed — Ollama & local models

These are **not billed per token** — cost is GPU/host compute, not token
volume. Price them at **USD 0/1M tokens** (token cost == 0) and surface them in
the report as `USD 0 self-hosted (compute-billed)`, NOT as `unknown`/`unpriced`.

| Model (match prefix)        | Input ($/1M) | Output ($/1M) | Class           |
|-----------------------------|--------------|---------------|-----------------|
| llama3.1 (8b / 70b / 405b)  |  0.00        |  0.00         | self-hosted     |
| mistral / mistral-small     |  0.00        |  0.00         | self-hosted     |
| orca-mini                   |  0.00        |  0.00         | self-hosted     |
| deepseek (deepseek-llm-*)   |  0.00        |  0.00         | self-hosted     |
| gpt-oss-*                   |  0.00        |  0.00         | self-hosted     |
| genai-demo / genai-model    |  0.00        |  0.00         | self-hosted/demo |

#### Resolution order (W-cost / W-econ apply this exactly)

1. **Priced match** (prefix/contains, case-insensitive) → multiply tokens by
   the rate. Record the model under `priced_models`.
2. **Self-hosted match** → token cost = **USD 0**, but the model and its token
   volume STILL appear in every table, tagged `USD 0 self-hosted
   (compute-billed)`. Add to `selfhosted_models`. This is NOT an
   `unknown_pricing` gap.
3. **No match at all** → token cost EXCLUDED from the $ total, but the model
   and its token volume are STILL surfaced in the per-model table tagged
   `UNPRICED — token cost excluded`, AND recorded in `gaps` as
   `unknown_pricing: <model>`. Surface these explicitly in a dedicated
   "Unpriced Models" sub-block so the reader sees coverage.

> **CRITICAL (RULE 10 interaction):** a tenant of entirely self-hosted /
> unknown models must NOT collapse total spend to `USD 0` and trip the
> abort/empty-report path. `USD 0 self-hosted` is a VALID, meaningful total
> (it means token cost is genuinely zero and the spend lives in compute).
> The report ALWAYS renders the token-volume tables even when the dollar
> total is `USD 0` or partially unpriced. Never abort on a `USD 0` total.

> **VALIDATED 2026-06-03 (live `demo` tenant):** observed models were
> `genai-demo` (self-hosted/demo), `titan-embed-text-v1` (Bedrock priced),
> `textembedding-gecko@001` (Vertex priced), `mistral-small:22b`,
> `llama3.1:8b/405b`, `orca-mini:3b` (self-hosted), `gpt-4o`,
> `text-embedding-3-*`, `text-embedding-ada-002` (OpenAI priced),
> `gemini-*` (Vertex priced). The OLD gpt-4/gpt-3.5/claude-only table had
> ZERO overlap with the dominant models and forced a `USD 0` total → RULE-10
> abort. This table now covers all of them.

> **VALIDATE** before any customer-shareable artifact: re-check each
> provider's public pricing page (OpenAI, Anthropic, AWS Bedrock, Google
> Vertex). Customer LLM spend is a sensitive number — a stale price
> multiplied across millions of tokens produces a misleading total. If
> pricing has shifted, update the table here AND in the W-cost / W-econ
> worker prompts before the run.

---

## EXECUTION PROTOCOL

### Run logging (async, non-blocking)

Maintain a per-run timing log next to the report:
`LOG="AIOBS_{APP_SLUG}_{DATE}.log"`. At each phase boundary append ONE line,
backgrounded so it never gates execution:
```bash
{ printf '[%s] %s\n' "$(date -u +%FT%TZ)" "Phase X — <one-line summary>" >> "$LOG"; } &
```
Log: START, Phase 0b complete (DispatchContext summary), Phase 1 complete
(per-worker one-liners), Absence-Gate result, report written, RUN COMPLETE.

---

### Phase 0a: Argument Parser

```
APP:"<value>"         → APP_NAME (required)
FROM:"<iso>"          → WINDOW.from (required)
TO:"<iso>"            → WINDOW.to (required)
--framework <name>    → FRAMEWORK in {langgraph, crewai, openai, anthropic} (required)

FLAGS:
  -clean         → CLEAN_MODE = true
  --pdf          → PDF_MODE = true (default FALSE — Markdown is canonical)
                   • legacy alias: pdf=true  → PDF_MODE = true
                   • pdf=false → no-op (MD-only is already the default)
  appendix=true  → FULL_APPENDIX = true (default false)

VALIDATION:
  - Missing APP / FROM / TO / --framework → print usage and exit
  - FROM >= TO → print usage and exit
  - Window > 14 days → warn (W-cost may exceed orchestrator memory budget)
```

---

### Phase 0-auth: PRE-WARM the dtctl session (Orchestrator)

**FIRST action of the whole run — before any data query and before any
dispatch:**
```
dtctl auth refresh
```
This is a REFRESH, not a login — silent, uses the on-disk refresh token
(`DTCTL_TOKEN_STORAGE=file`, harness env, never the Keychain). It guarantees a
fresh access token up front, and the parallel workers inherit one fresh token
instead of racing concurrent refreshes.

- `dtctl auth refresh` succeeds → proceed to Phase 0b.
- It fails (no/expired refresh token) → ONLY THEN run once:
  `dtctl auth login --plain --safety-level readonly`.

**Workers never authenticate.** If a worker returns `<gap: auth>`, the
orchestrator refreshes once and re-dispatches only that worker.

> **AUTH HARDENING (NON-NEGOTIABLE — added 2026-06-03):**
> - **Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`.** On any auth-shaped error a worker returns `<gap: auth>` and STOPS immediately. Authentication is exclusively the orchestrator's job, done serially — never inside a subagent.
> - **The orchestrator runs `dtctl auth login` at most ONCE per run, and ONLY as an interactive browser flow.** NEVER spawn it from a subagent, in the background, or in parallel. Concurrent / non-interactive `dtctl auth login` attempts corrupt the on-disk token store (observed 2026-06-03: the `demo` context OAuth session was wiped and flipped to a missing "platform token", blocking every query).
> - If `dtctl auth refresh` fails AND an interactive login cannot be completed in the current context (headless / subagent / automation), the orchestrator **STOPS with an "Auth unavailable" banner** and surfaces it to the user. It does NOT retry logins in a loop and NEVER lets workers attempt recovery.

---

### Phase 0b: Context Bootstrap (Sequential — Orchestrator)

Build the **DispatchContext** that all workers receive.

#### Step 0b.0 — Substrate Probe (runs ONCE, before everything else)

> **Why:** the orchestrator bootstrap must NOT assume `APP` is a service. On
> this tenant the obvious anchor `genai-demo` is a **`gen_ai.request.model`
> value** (76k+ calls), not a SERVICE — `smartscapeNodes SERVICE contains
> "genai"` returns 0 rows. The probe self-heals against this drift: it
> resolves `APP` against BOTH substrates at runtime and binds whichever
> matches, instead of dying on a stale single-substrate assumption.

Run these **two cheap probes in parallel** (one message), then select:

**Probe A — APP as a SERVICE** (candidate substrate #1):

<!-- VALIDATED 2026-06-03 via dtctl: APP:"ai-travel-advisor-agent-test" → SERVICE-E2063290CDC0624B (1 row) -->
```dql
smartscapeNodes SERVICE
| filter contains(name, "{APP_NAME}")
| fields id, name
| limit 5
```

**Probe B — APP as a `gen_ai.request.model` VALUE** (candidate substrate #2):

<!-- VALIDATED 2026-06-03 via dtctl: APP:"genai-demo" → 76,890 calls (0 SERVICE rows) -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter gen_ai.request.model == "{APP_NAME}"
| summarize calls = count(),
            in_tok = sum(gen_ai.usage.input_tokens),
            out_tok = sum(gen_ai.usage.output_tokens)
```

**Bind rule — CANDIDATE LIST ordered [service, model]:**
1. **Probe A returns ≥1 SERVICE row** → `APP_BIND_MODE = "service"`,
   `APP_SERVICE_ID = <id>`, `APP_DISPLAY_NAME = <name>`. (If multiple, pick
   the highest-traffic SERVICE in the window — `fetch spans … | summarize
   count() by dt.smartscape.service | sort … desc` — and document in
   Appendix A.) The workers' span filter uses the standard
   `dt.smartscape.service in [smartscapeNodes SERVICE | filter id == …]`.
2. **Probe A empty AND Probe B `calls > 0`** → `APP_BIND_MODE = "model"`,
   `APP_MODEL_VALUE = "{APP_NAME}"`, `APP_SERVICE_ID = null`,
   `APP_DISPLAY_NAME = "{APP_NAME} (model)"`. The workers' span filter
   becomes `filter gen_ai.request.model == "{APP_MODEL_VALUE}"`
   (NO smartscape subquery — there is no service to scope to).
3. **Both empty** → emit the "APP Entity Not Found" banner (see ERROR
   HANDLING). Absence is gated, never assumed.

Emit a one-line probe summary to the run log, e.g.
`Probe: APP_BIND_MODE=model (genai-demo → 76,890 calls); service probe=0 rows`
or `Probe: APP_BIND_MODE=service (ai-travel-advisor-agent-test → SERVICE-E2063…)`.

> **`SPAN_SCOPE_FILTER` — single source of truth for all workers.** Phase
> 0b.0 emits a `SPAN_SCOPE_FILTER` string into the DispatchContext that EVERY
> worker (and every Phase-0b/1.5 query) substitutes verbatim instead of
> hardcoding the smartscape subquery:
> - `service` mode → `filter dt.smartscape.service in [ smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id ]`
> - `model` mode   → `filter gen_ai.request.model == "{APP_MODEL_VALUE}"`
>
> Wherever the worker queries below show the smartscape subquery, read it as
> `{SPAN_SCOPE_FILTER}`.

#### Step 0b.1 — (folded into 0b.0)

APP resolution is now performed in **Step 0b.0** above (dual service/model
bind). `mcp__dynatrace__find_entity_by_name` remains an OPTIONAL accelerator
for the service path ONLY — if the MCP is registered and returns a SERVICE,
use it to populate `APP_SERVICE_ID`; otherwise Probe A (smartscapeNodes) is
the primary, always-available path (MCP is unavailable on this tenant). The
MCP is never a hard dependency.

#### Step 0b.2 — Validate window has spans

Uses `{SPAN_SCOPE_FILTER}` from Phase 0b.0 (service-bind OR model-bind).

<!-- VALIDATED 2026-06-03: model-bind genai-demo → 76,890 spans; service-bind SERVICE-E2063290CDC0624B → 4,805 spans (3d window) -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| {SPAN_SCOPE_FILTER}
| summarize span_count = count()
```

If `span_count == 0`, abort with "no spans in window for {APP_NAME}" — most
common cause is wrong APP name or wrong tenant. (In model-bind mode this is
already proven > 0 by Probe B, so it can only fire if the window changed.)

#### Step 0b.3 — Detect framework + sample span schema

Uses `{SPAN_SCOPE_FILTER}` from Phase 0b.0.

<!-- VALIDATED 2026-06-03: gen_ai.system is null or "Azure" on this tenant — NEVER "openai"/"anthropic". -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| {SPAN_SCOPE_FILTER}
| filter isNotNull(gen_ai.request.model) or isNotNull(gen_ai.system)
| limit 5
| fields name, gen_ai.system, gen_ai.request.model,
         langgraph.node, crewai.task.id, crewai.agent,
         attributes
```

From the sample row(s) confirm:
- `gen_ai.system` vs `--framework`: **warn-and-PROCEED on ANY mismatch,
  including `gen_ai.system == null`.** This is the COMMON case, not an error.
  Validated 2026-06-03: tenant `gen_ai.system` is only `null` or `"Azure"` —
  it NEVER equals `"openai"`/`"anthropic"`/`"langgraph"`/`"crewai"`. The
  `--framework` flag is authoritative for non-`gen_ai.*` attribute selection
  (per QUALITY RULE 2). A mismatch (or null) NEVER aborts and NEVER zeroes
  out the analysis — record it in Appendix A and continue.
- For `--framework langgraph`: `langgraph.node` is present on at least one
  sample. If absent, warn (the app may emit only `gen_ai.*`) and proceed —
  W-latency/W-loops fall back to bare span `name` for the step key.
- For `--framework crewai`: same — `crewai.task.id`/`crewai.agent` absence is
  a warn-and-proceed, not an abort.
- Capture the FULL `attributes` map and grep for `request_attribute.*` keys —
  this discovers `feature.name`, `session.id`, `conversation.id`, etc., which
  W-loops and W-econ key off.

If no `gen_ai.*` attributes are present (model-bind mode guarantees they
are), abort with "service is not LLM-instrumented (no OpenLLMetry spans
found in window)" — but ONLY after the Absence-Gate-style unfiltered
confirmation, never on a single empty scoped query.

#### Step 0b.4 — Build DispatchContext

```json
{
  "APP_NAME": "...",
  "APP_DISPLAY_NAME": "...",
  "APP_BIND_MODE": "service | model",
  "APP_SERVICE_ID": "SERVICE-XXXX | null (null in model-bind mode)",
  "APP_MODEL_VALUE": "null (set to the model value in model-bind mode, e.g. genai-demo)",
  "SPAN_SCOPE_FILTER": "filter dt.smartscape.service in [ smartscapeNodes SERVICE | filter id == \"SERVICE-XXXX\" | fields id ]   — OR —   filter gen_ai.request.model == \"genai-demo\"",
  "FRAMEWORK": "langgraph|crewai|openai|anthropic",
  "FRAMEWORK_SYSTEM_MISMATCH": "false | true (warn-and-proceed; gen_ai.system was null/Azure)",
  "WINDOW": { "from": "ISO", "to": "ISO" },
  "DETECTED_GEN_AI_SYSTEM": "null | Azure | openai | anthropic | bedrock | ...",
  "CUSTOM_REQUEST_ATTRIBUTES": ["feature.name", "session.id", "conversation.id"],
  "SAMPLE_MODELS_SEEN": ["genai-demo", "titan-embed-text-v1", "mistral-small:22b"],
  "UNPRICED_MODELS": [],
  "CLEAN_MODE": false,
  "FULL_APPENDIX": false
}
```

> **Workers MUST use `SPAN_SCOPE_FILTER` verbatim** for every span query. In
> model-bind mode there is no `APP_SERVICE_ID` — the smartscape subquery is
> replaced by a direct `gen_ai.request.model ==` predicate. Worker queries
> below that still show the smartscape subquery are illustrative of the
> service-bind shape only.

---

### Phase 0c: Reference Strategy

dt-ai-obs is reference-driven. Each worker reads ONE targeted reference file
at start, derives query patterns from it, then applies the dt-ai-obs scoping
below. Orchestrator inline DQL is limited to Phase 0b bootstrap and Phase 1.5
Absence-Gate confirmations.

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

**Dispatch ALL FOUR workers in ONE message** with multiple Agent calls
(`subagent_type: Explore`). True concurrency — sequential dispatch defeats
the architecture.

> **SCOPING — read this before the worker queries below.** Every worker
> query that shows
> `filter dt.smartscape.service in [ smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id ]`
> is illustrating the **service-bind** shape only. Workers MUST substitute
> the DispatchContext `{SPAN_SCOPE_FILTER}` verbatim — which is that
> smartscape subquery in service-bind mode, OR
> `filter gen_ai.request.model == "{APP_MODEL_VALUE}"` in model-bind mode.
> In model-bind mode there is NO service to scope to. This applies to W-latency,
> W-cost, W-loops, and W-econ identically.

Expected total: bootstrap + worker fan-out (parallel) + synthesis ≈ 6–9 min
at medium effort.

**Worker-failure handling — NO silent serial fallback.** If a worker returns
gaps, re-dispatch THAT ONE worker ONCE (fresh Agent call, note what to fix);
if it still fails, record a `⚠️ gap` in the report. The orchestrator NEVER
runs the worker's queries inline.

#### Worker Prompt Template (shared shell)

```
You are an LLM-observability worker for dt-ai-obs.

ANTI-FABRICATION (READ FIRST — non-negotiable):
If your dtctl queries return no usable data — an auth error, an `{ok:false}` error envelope, an
empty result, or no successfully-executed query at all — you have NOTHING to report. Return your
gap marker (`<gap: ...>`) and STOP. NEVER write a metric, count, latency, rate, or ANY number you
did not read directly from a successful query result. Inventing plausible-looking numbers is the
single worst failure mode of this skill: a `<gap>` is always correct and safe; a fabricated number
is never acceptable. If you cannot point to the query result a number came from, do not emit it.

STEP 1 — READ YOUR REFERENCE (first action, before any query):
Read ONLY the reference file(s) listed in REFERENCE FILES. Read each once.
This is the source of DQL truth — do NOT derive queries from training
knowledge or guess field names.

STEP 2 — RUN ALL QUERIES IN ONE PARALLEL BATCH:
Issue ALL independent queries in ONE message (multiple `dtctl query` tool
calls in parallel). Do not interleave file reads and queries.

ENTITY MODEL — CRITICAL:
- NEVER use dt.entity.service == to filter spans. Use the
  {SPAN_SCOPE_FILTER} string from the DispatchContext VERBATIM — it is the
  single source of truth for scoping and is one of:
    • service-bind: filter dt.smartscape.service in [smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id]
    • model-bind:   filter gen_ai.request.model == "{APP_MODEL_VALUE}"
  Do NOT hardcode the smartscape subquery — in model-bind mode there is no
  service to scope to (APP_SERVICE_ID is null).
- Time filter on spans: start_time >= toTimestamp("{WINDOW.from}") AND start_time <= toTimestamp("{WINDOW.to}")
- ALIAS every bin (`ts = bin(start_time, 1h) | sort ts asc`). NEVER backticks.
- SINGLE QUOTES around the DQL when calling dtctl (double-quote breaks `$` and `` ` ``).
- Prefix EVERY dtctl call with `DTCTL_TOKEN_STORAGE=file` (workers do NOT
  inherit the harness env block).

AUTH RULES:
- Workers NEVER run `dtctl auth login` OR `dtctl auth refresh`. On auth error,
  return `<gap: auth>` and STOP — never attempt any auth recovery. An auth error
  is NEVER evidence of "no data".

ABSENCE RULES:
- ERROR ≠ ABSENCE. A query that errored proves nothing.
- A scoped 0-row result must be confirmed by an UNFILTERED query before
  concluding "no data". A claim of absence WITHOUT proof is invalid — the
  orchestrator will reject it and re-dispatch you.

DISPATCH CONTEXT:
{DISPATCH_CONTEXT_JSON}

QUERIES TO RUN:
{QUERY_INSTRUCTIONS_FOR_THIS_WORKER}

RETURN only the PhaseResult JSON shape. Nothing else. Cap at ~4KB.
```

---

#### W-latency — per-prompt latency

**Reference:** `~/.agents/skills/dt-obs-tracing/references/performance-analysis.md`
(percentile / makeTimeseries patterns) + `request-attributes.md` (request-
level aggregation).

**Goal:** p50 / p95 / p99 latency for LLM-call spans, grouped by
`gen_ai.request.model` and the framework-step attribute
(`langgraph.node` / `crewai.task.id` / `crewai.agent` for agentic
frameworks; bare `name` for direct SDK).

**Queries (ALL in one parallel batch):**

1. **Per-model latency distribution**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize calls = count(),
            p50_ms = percentile(duration, 50) / 1000000,
            p95_ms = percentile(duration, 95) / 1000000,
            p99_ms = percentile(duration, 99) / 1000000,
            max_ms = max(duration) / 1000000,
            by: { gen_ai.request.model }
| sort p95_ms desc
```

2. **Per-step latency (framework-specific group key)** — substitute the
   group-by field per FRAMEWORK:
   - langgraph: `by: { langgraph.node, gen_ai.request.model }`
   - crewai:    `by: { crewai.task.id, crewai.agent, gen_ai.request.model }`
   - openai / anthropic: `by: { name, gen_ai.request.model }` (span name is
     the operation, e.g. `openai.chat`)

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize calls = count(),
            p50_ms = percentile(duration, 50) / 1000000,
            p95_ms = percentile(duration, 95) / 1000000,
            p99_ms = percentile(duration, 99) / 1000000,
            by: { {{FRAMEWORK_STEP_KEY}}, gen_ai.request.model }
| sort p95_ms desc
| limit 25
```

3. **Latency trend over the window**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize p95_ms = percentile(duration, 95) / 1000000,
            calls = count(),
            by: { ts = bin(start_time, 1h), gen_ai.request.model }
| sort ts asc
```

**PhaseResult shape:**
```json
{
  "per_model_latency": [
    { "model": "...", "calls": 0, "p50_ms": 0, "p95_ms": 0, "p99_ms": 0, "max_ms": 0 }
  ],
  "per_step_latency": [
    { "step": "...", "model": "...", "calls": 0, "p50_ms": 0, "p95_ms": 0, "p99_ms": 0 }
  ],
  "latency_trend": [{ "time_bucket": "...", "model": "...", "p95_ms": 0, "calls": 0 }],
  "slowest_step": { "step": "...", "model": "...", "p95_ms": 0 },
  "gaps": []
}
```

---

#### W-cost — token cost outliers

**Reference:** `~/.agents/skills/dt-obs-tracing/references/request-attributes.md`
(request-level aggregation, custom attribute access).

**Goal:** cost per span / per model / per step. Identify the top-N most
expensive prompts and the total spend in the window.

**Pricing model — EMBED in the worker prompt** (copy the table from the
"Pricing Table" section above; workers do NOT read it from disk).

**Queries (ALL in one parallel batch):**

1. **Token totals + estimated cost per model** — compute cost in the worker
   AFTER the query returns (DQL can't easily do a model-keyed lookup, and
   embedding a giant `if(model=="...", X, if(model=="...", Y, ...))` is
   fragile). The query returns `input_tokens_sum`, `output_tokens_sum`,
   `call_count` per model; the worker multiplies by the pricing table.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize calls = count(),
            input_tokens_sum = sum(gen_ai.usage.input_tokens),
            output_tokens_sum = sum(gen_ai.usage.output_tokens),
            avg_input = avg(gen_ai.usage.input_tokens),
            avg_output = avg(gen_ai.usage.output_tokens),
            by: { gen_ai.request.model }
| sort input_tokens_sum desc
```

2. **Top-N most expensive single calls** — outlier detection. Surface the
   spans with the largest token footprint; these often indicate a runaway
   prompt or an unbounded context-window concatenation.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.usage.input_tokens) or isNotNull(gen_ai.usage.output_tokens)
| fieldsAdd total_tokens = coalesce(gen_ai.usage.input_tokens, 0) + coalesce(gen_ai.usage.output_tokens, 0)
| sort total_tokens desc
| limit 25
| fields trace.id, span.id, start_time, gen_ai.request.model,
         gen_ai.usage.input_tokens, gen_ai.usage.output_tokens, total_tokens, duration
```

3. **Token spend trend**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize input_tokens = sum(gen_ai.usage.input_tokens),
            output_tokens = sum(gen_ai.usage.output_tokens),
            by: { ts = bin(start_time, 1h), gen_ai.request.model }
| sort ts asc
```

**Worker post-processing (compute cost from the embedded pricing table —
follow the 3-tier resolution order):**
- `cost_usd = (input_tokens_sum / 1e6) * input_price + (output_tokens_sum / 1e6) * output_price`
- Match each `gen_ai.request.model` PREFIX/CONTAINS (case-insensitive),
  stripping provider prefixes (`bedrock/`, `anthropic.`), version suffixes
  (`@001`, `:22b`, `:8b`, `:405b`, `-002`), and date stamps:
  1. **Priced match** → multiply tokens by the rate; tag `priced`.
  2. **Self-hosted match** (llama / mistral / orca-mini / genai-demo /
     genai-model) → `cost_usd = 0`; tag `USD 0 self-hosted (compute-billed)`;
     add to `selfhosted_models`. The row's TOKEN VOLUME still appears in the
     per-model table. This is NOT an `unknown_pricing` gap.
  3. **No match** → token cost EXCLUDED from `total_cost_usd`, but the row
     (with token volume) STILL appears tagged `UNPRICED — token cost
     excluded`; add to `gaps` as `unknown_pricing: <model>` AND to
     `unpriced_models`.
- **Never let `USD 0` abort.** A window of entirely self-hosted/unknown models
  yields `total_cost_usd = 0.0` — that is a VALID result (token cost is
  genuinely zero / lives in compute), NOT an empty-data condition. Always
  return the populated token tables. See QUALITY RULE 10.

**PhaseResult shape:**
```json
{
  "total_cost_usd": 0.0,
  "total_input_tokens": 0,
  "total_output_tokens": 0,
  "per_model_cost": [
    { "model": "...", "pricing_class": "priced | selfhosted | unpriced",
      "calls": 0, "input_tokens": 0, "output_tokens": 0,
      "input_cost_usd": 0.0, "output_cost_usd": 0.0, "total_cost_usd": 0.0 }
  ],
  "selfhosted_models": ["llama3.1:8b", "mistral-small:22b", "genai-demo"],
  "unpriced_models": [],
  "top_n_expensive_spans": [
    { "trace_id": "...", "span_id": "...", "start_time": "...", "model": "...",
      "input_tokens": 0, "output_tokens": 0, "total_tokens": 0,
      "estimated_cost_usd": 0.0, "pricing_class": "priced | selfhosted | unpriced" }
  ],
  "cost_trend": [{ "time_bucket": "...", "model": "...", "cost_usd": 0.0 }],
  "pricing_table_used_as_of": "est. 2026",
  "gaps": []
}
```

---

#### W-loops — agent-loop detection

**Reference:** `~/.agents/skills/dt-obs-tracing/references/request-attributes.md`
(request aggregation, custom attribute access).

**Goal:** within a single session / conversation, detect repeated identical
or near-identical prompts firing within a short window. This is the
characteristic signature of an agent stuck in a loop (langgraph reflexion
cycle, crewai task retry, or a generic ReAct loop that keeps re-asking).

**Detection strategy:**
- Group by session / conversation ID (`request_attribute.session.id`,
  `request_attribute.conversation.id`, or fall back to `trace.id` if neither
  is present).
- Within each group, hash the user-turn content (`gen_ai.prompt.<n>.content`
  where role == `user`) and count duplicate hashes within a rolling
  N-minute window (default N = 5).
- A session with >= 3 identical user prompts in <= 5 minutes is flagged.

**Queries (ALL in one parallel batch):**

1. **Repeated prompts within sessions** — use Dynatrace `hash_md5()` (or
   `toString` + group; verify which is available in the tenant). The
   reference file `request-attributes.md` documents the safe pattern.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.prompt.0.content)
| fieldsAdd session_key = coalesce(request_attribute.session.id,
                                    request_attribute.conversation.id,
                                    toString(trace.id))
| fieldsAdd prompt_hash = hash_md5(gen_ai.prompt.0.content)
| summarize repeats = count(),
            first_seen = takeMin(start_time),
            last_seen = takeMax(start_time),
            by: { session_key, prompt_hash, gen_ai.request.model }
| filter repeats >= 3
| fieldsAdd window_minutes = (last_seen - first_seen) / 60000000000
| filter window_minutes <= 5
| sort repeats desc
| limit 25
```

2. **Per-session call volume distribution** — flag sessions with anomalously
   high LLM-call counts; these are loop candidates even if the prompt text
   varies slightly across iterations.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| fieldsAdd session_key = coalesce(request_attribute.session.id,
                                    request_attribute.conversation.id,
                                    toString(trace.id))
| summarize llm_calls = count(),
            distinct_models = count_distinct(gen_ai.request.model),
            duration_min = (takeMax(start_time) - takeMin(start_time)) / 60000000000,
            by: { session_key }
| sort llm_calls desc
| limit 25
```

3. **Per-step loop signature (framework-specific)** — for langgraph, detect a
   single `langgraph.node` firing repeatedly inside one trace:

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(langgraph.node)
| summarize node_hits = count(),
            by: { trace.id, langgraph.node }
| filter node_hits >= 5
| sort node_hits desc
| limit 25
```

For crewai, swap `langgraph.node` → `crewai.task.id`. For openai/anthropic
(no framework), this third query is replaced by a duplicate-prompt-by-
`trace.id` query (group by trace + prompt_hash).

**PhaseResult shape:**
```json
{
  "repeated_prompt_loops": [
    { "session_key": "...", "model": "...", "repeats": 0,
      "window_minutes": 0, "first_seen": "...", "last_seen": "..." }
  ],
  "high_volume_sessions": [
    { "session_key": "...", "llm_calls": 0, "distinct_models": 0, "duration_min": 0 }
  ],
  "step_loop_signatures": [
    { "trace_id": "...", "step": "...", "node_hits": 0 }
  ],
  "loop_count_total": 0,
  "gaps": []
}
```

---

#### W-econ — per-feature unit economics ($/feature)

**Reference:** `~/.agents/skills/dt-obs-tracing/references/request-attributes.md`
(custom `request_attribute.*` access) +
`~/.agents/skills/dt-obs-services/references/service-metrics.md` (RED metric
patterns for per-feature request counts).

**Goal:** group LLM cost by a `feature.name` (or fallback) custom request
attribute and surface the top-cost features. Combined with feature request
counts, derives a per-feature unit cost — "$/conversation",
"$/summarization", etc.

**Discovery rule:** the orchestrator's Phase 0b sample-span query lists
detected custom attributes. W-econ chooses the first one of:
`request_attribute.feature.name`, `request_attribute.endpoint.name`,
`request_attribute.feature`, falling back to bare span `name` if none are
present. The chosen key goes into the DispatchContext as `FEATURE_KEY` and
W-econ keys off it.

> **No-feature-key degradation (VALIDATED 2026-06-03):** on this tenant, in
> model-bind mode (`genai-demo`) NONE of the custom attributes are populated
> and span `name` is also null — so `FEATURE_KEY` cannot resolve to anything
> meaningful (every span lands in one null bucket). When that happens:
> - set `FEATURE_KEY = "gen_ai.request.model"` as the LAST-RESORT key (model
>   IS a legitimate economic dimension), and
> - set `per_feature_economics[].feature` accordingly, and
> - record `feature_key_used: "gen_ai.request.model (no feature attribute /
>   span name present)"` so the report is honest about the fallback.
> Never emit a single-row "feature: null" table — that reads as a bug. This
> keeps the per-feature section meaningful even with zero custom
> instrumentation.

**Queries (ALL in one parallel batch):**

1. **Token + call volume by feature**

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter isNotNull(gen_ai.request.model)
| summarize calls = count(),
            input_tokens = sum(gen_ai.usage.input_tokens),
            output_tokens = sum(gen_ai.usage.output_tokens),
            by: { {{FEATURE_KEY}}, gen_ai.request.model }
| sort input_tokens desc
| limit 25
```

2. **Feature request rate (RED-style)** — count of REQUEST root spans per
   feature, to derive cost-per-request.

<!-- VALIDATE: run via dtctl query against tenant before first production use -->
```dql
fetch spans, from:toTimestamp("{WINDOW.from}"), to:toTimestamp("{WINDOW.to}")
| filter dt.smartscape.service in [
    smartscapeNodes SERVICE | filter id == "{APP_SERVICE_ID}" | fields id
  ]
| filter request.is_root_span == true
| summarize requests = count(),
            by: { {{FEATURE_KEY}} }
| sort requests desc
```

**Worker post-processing:**
- Compute per-feature cost from the embedded pricing table using the SAME
  3-tier resolution as W-cost (priced → multiply; self-hosted → USD 0; unknown
  → exclude from $ but keep token volume + record `unknown_pricing`). A
  feature served entirely by self-hosted models has a legitimate `USD 0` token
  cost — surface it, do not drop it.
- Compute `cost_per_request_usd = total_cost_usd / requests` per feature
  (guard requests==0 → null, not divide-by-zero).
- Rank features by total cost AND by cost-per-request (the second often
  surfaces a low-volume but expensive feature that drives margin).

**PhaseResult shape:**
```json
{
  "feature_key_used": "feature.name | endpoint.name | name",
  "per_feature_economics": [
    { "feature": "...", "calls": 0, "input_tokens": 0, "output_tokens": 0,
      "total_cost_usd": 0.0, "requests": 0, "cost_per_request_usd": 0.0 }
  ],
  "top_cost_feature": { "feature": "...", "total_cost_usd": 0.0 },
  "top_cpr_feature": { "feature": "...", "cost_per_request_usd": 0.0 },
  "gaps": []
}
```

---

### Phase 1.5: Absence Gate (Orchestrator)

Before trusting any worker result, scan every PhaseResult for absence claims
("no LLM spans", "feature.name not present", "0 loops detected", etc.). For
EACH such claim:

1. Require the worker's proof (the unfiltered confirmation query + zero-row
   result). If missing, the claim is invalid.
2. Run ONE cheap unfiltered confirmation query yourself:
   - "no LLM spans": `fetch spans, <window> | filter dt.smartscape.service in
     [...APP...] | summarize count() by isNotNull(gen_ai.request.model)`.
   - "feature.name not present": `fetch spans, <window> | filter <APP> |
     summarize count() by isNotNull(request_attribute.feature.name)`.
   - "0 loops": this is a legitimate finding — confirm only that
     `repeated_prompt_loops` AND `high_volume_sessions` were both empty;
     if either had rows, loops exist and the worker mis-ranked them.
3. Only after broad confirmation may "absent" enter the report.

---

### Phase 2: Synthesis (Orchestrator)

After all four workers return and Absence Gate clears, synthesize a ranked
findings table. There is no hypothesis graph here (this is a Tier-2 brief,
not a forensic investigation) — just a priority-ranked list:

```
Rank findings by severity:
  - CRITICAL: any p95 > 30s on a user-facing step (W-latency)
  - CRITICAL: top-cost feature > 60% of total window spend (W-econ)
  - HIGH:     loop_count_total > 0 (W-loops)
  - HIGH:     top-N expensive single span > USD 1.00 each (W-cost)
  - MEDIUM:   high_volume_sessions with > 50 LLM calls (W-loops)
  - MEDIUM:   per-step p95 > 2x per-model p95 (W-latency)
  - LOW:      everything else worth noting
```

The synthesis table feeds the Executive Summary's "Top Concerns" block and
the Recommended Actions section.

---

### Phase 3: Davis CoPilot Synthesis (Orchestrator)

Pass the ranked findings (NOT raw PhaseResults) to Davis CoPilot. Ask for
remediation guidance — specifically the three levers that move LLM economics:

```
Tool: mcp__dynatrace__chat_with_davis_copilot

Input text:
LLM APPLICATION OBSERVABILITY BRIEF: {APP_DISPLAY_NAME}
FRAMEWORK: {FRAMEWORK}
WINDOW: {WINDOW.from} to {WINDOW.to}
TOTAL SPEND: ${TOTAL_COST_USD}
TOTAL CALLS: {TOTAL_CALLS}

RANKED FINDINGS:
1. [CRITICAL] {finding}
2. [HIGH]     {finding}
3. ...

QUESTIONS:
1. For each finding, recommend a specific remediation. Focus on:
   - CHEAPER-MODEL SUBSTITUTION (which steps could move from gpt-4 to
     gpt-3.5-turbo, or claude-opus to claude-sonnet/haiku, with what
     quality trade-off)
   - PROMPT-CACHING OPPORTUNITY (which prompts have a stable prefix that
     would benefit from Anthropic prompt caching or OpenAI cached input)
   - LOOP-BREAK STRATEGY (what guardrail would catch the detected loops —
     iteration cap, semantic-duplicate detection, or framework checkpoint
     limit)
2. For each Top-Cost Feature: would batching, RAG, or output-token capping
   yield the largest savings?
3. Confidence in your assessment, and what additional telemetry would
   increase confidence (e.g. capturing prompt content hashes natively).
```

Davis CoPilot's response feeds the "Recommended Actions" and "Davis
Synthesis" sections.

---

### Phase 4: Report Generation

If CLEAN_MODE, build the sanitization map first (identical to dt-rca Phase
1.14 — read that section for full rules: replace APP names, session IDs,
conversation IDs, user IDs, trace IDs, tenant URLs, any custom-attribute
values that could identify a customer; preserve model names, framework
names, generic span names) and compose with sanitized names from the start.

**Sanitization, URL format, Mermaid/ASCII diagram rules, PDF generation, and
filename rules are identical to `/dt-rca`. Read
`~/.claude/skills/dt-rca/SKILL.md` sections "Phase 1.14", "Phase 3",
"Phase 4", "PDF DIAGRAM RULES", "MERMAID SYNTAX RULES", and "DIAGRAM SIZING
RULES" — all apply unchanged.**

Emit the report EXACTLY in the section order below. Each fact has ONE
canonical home; never restate the same data in another section.

```markdown
# dt-ai-obs LLM Application Observability Brief
## Application {APP_DISPLAY_NAME} — {FRAMEWORK}

**Generated:** {CURRENT_DATE}
**Analyst:** Claude {MODEL} (Dynatrace MCP Integration)
**Environment:** {TENANT_URL}
**Window:** {WINDOW.from} → {WINDOW.to}  |  **Framework:** {FRAMEWORK}
**Workers:** W-latency · W-cost · W-loops · W-econ

---

## Executive Summary

### Headline
[One sentence. Lead with total spend + top concern. Example:
"Window spend was USD 1,247 across 28,114 LLM calls; top concern is a CRITICAL
agent-loop pattern in the support-agent session-router consuming ~USD 340 of
that total."]

### Window Totals
| Metric                | Value           |
|-----------------------|-----------------|
| Total LLM Calls       | {TOTAL_CALLS}   |
| Total Input Tokens    | {INPUT_TOKENS}  |
| Total Output Tokens   | {OUTPUT_TOKENS} |
| Estimated Spend (USD) | ${TOTAL_COST}   |
| Distinct Models Used  | {N}             |
| Sessions Observed     | {N}             |

### Top Concerns
| Priority | Finding | Evidence |
|----------|---------|----------|
[3-5 rows from Phase 2 ranked findings]

### Recommended Immediate Action
| Priority | Action | Lever | Est. Impact |
|----------|--------|-------|-------------|
[From Davis CoPilot for the top CRITICAL/HIGH finding. Lever ∈
{model substitution, prompt caching, loop break, batching, output cap}.]

---

## Latency Analysis

### Per-Model Latency
*Source: W-latency*
| Model | Calls | p50 (ms) | p95 (ms) | p99 (ms) | Max (ms) |
|-------|-------|----------|----------|----------|----------|

### Per-Step Latency (Top 10)
*Group key: {langgraph.node | crewai.task.id | crewai.agent | span name}*
| Step | Model | Calls | p50 (ms) | p95 (ms) | p99 (ms) |
|------|-------|-------|----------|----------|----------|

### Latency Trend
```mermaid
[time-series chart of p95 by model over the window — per dt-rca Mermaid
rules. ASCII-only in the PDF version.]
```

---

## Cost Top-N

### Per-Model Cost Breakdown
*Source: W-cost. Pricing table as of: {pricing_table_used_as_of}.*
*`Class` ∈ {priced, USD 0 self-hosted, UNPRICED}. UNPRICED rows show token
volume but are excluded from the dollar total.*
| Model | Class | Calls | Input Tokens | Output Tokens | Input Cost | Output Cost | Total |
|-------|-------|-------|--------------|---------------|------------|-------------|-------|

### Self-Hosted & Unpriced Models
*Token volume is real; dollar cost is USD 0 (self-hosted/compute-billed) or
excluded (unpriced). Surfaced for coverage transparency — required whenever
either list is non-empty.*
| Model | Class | Calls | Total Tokens | Note |
|-------|-------|-------|--------------|------|
[self-hosted rows: "compute-billed; token $ = 0".
 unpriced rows: "no price in table; add to pricing table to include in total".
 If BOTH lists empty: "All observed models are priced." ]

### Top-25 Most Expensive Single Calls
| Trace ID | Time | Model | Input Tokens | Output Tokens | Total Tokens | Est. Cost (USD) |
|----------|------|-------|--------------|---------------|--------------|-----------------|
[Trace IDs are sanitized in CLEAN_MODE. Each row links to the Dynatrace
trace in the live report.]

### Cost Trend
```mermaid
[cost-over-time chart, stacked by model]
```

---

## Loop Detections

### Repeated-Prompt Loops
*Source: W-loops. Threshold: ≥3 identical user prompts within ≤5 minutes
per session.*
| Session | Model | Repeats | Window (min) | First Seen | Last Seen |
|---------|-------|---------|--------------|------------|-----------|
[Session keys are sanitized in CLEAN_MODE.]

### High-Volume Sessions
| Session | LLM Calls | Distinct Models | Duration (min) |
|---------|-----------|-----------------|----------------|

### Step-Loop Signatures
*Single {langgraph.node | crewai.task.id} firing ≥5 times within one trace.*
| Trace ID | Step | Hits |
|----------|------|------|

**Loop classification:** {N} total loop instances; estimated wasted spend
${WASTED_COST_USD} (sum of token cost across detected loop iterations).

---

## Per-Feature Economics

*Group key:* `{FEATURE_KEY}` *(detected automatically in Phase 0b)*

### Top Features by Total Cost
| Feature | Calls | Input Tokens | Output Tokens | Total Cost (USD) |
|---------|-------|--------------|---------------|------------------|

### Top Features by Cost-per-Request
| Feature | Requests | Total Cost | Cost / Request (USD) |
|---------|----------|------------|----------------------|

[One paragraph synthesis: which feature is the spend driver, which feature
is the per-unit margin risk, and how the two rank against each other.]

---

## Recommended Actions

*Source: Phase 3 — Davis CoPilot synthesis on the ranked findings.*

### Model-Substitution Opportunities
| Step / Feature | Current Model | Proposed Model | Est. Savings | Quality Risk |
|----------------|---------------|----------------|--------------|--------------|

### Prompt-Caching Opportunities
| Step / Feature | Stable Prefix | Cache Hit Estimate | Est. Savings |
|----------------|---------------|--------------------|--------------|

### Loop-Break Guardrails
| Loop Pattern | Recommended Guardrail | Owner |
|--------------|------------------------|-------|

---

## Davis Synthesis

[Verbatim Davis CoPilot response, formatted. If Davis was unavailable, add
banner: "> Davis CoPilot synthesis unavailable. Recommendations derived from
cross-worker findings alone."]

---

## Appendix A: Investigation Details
| Field | Value |
|-------|-------|
| APP Anchor | {APP_NAME} |
| APP Bind Mode | {APP_BIND_MODE} (service / model) |
| APP Service ID | {APP_SERVICE_ID} (null in model-bind mode) |
| APP Model Value | {APP_MODEL_VALUE} (null in service-bind mode) |
| Framework | {FRAMEWORK} |
| Framework↔`gen_ai.system` | {match / MISMATCH — warn-and-proceed} |
| Window | {WINDOW.from} → {WINDOW.to} |
| Detected `gen_ai.system` | {DETECTED_GEN_AI_SYSTEM} (often null / Azure) |
| Custom Request Attributes Found | {CUSTOM_REQUEST_ATTRIBUTES} |
| Feature Key Used | {FEATURE_KEY} |
| Sample Models Seen | {SAMPLE_MODELS_SEEN} |
| Self-Hosted Models (USD 0) | {selfhosted_models} |
| Unpriced Models (excluded from $) | {unpriced_models} |
| Pricing Table As Of | {pricing_table_used_as_of} |
| Davis CoPilot | {Used / Unavailable} |

### dt-ai-obs Worker Telemetry
| Worker | Reference | Key Findings | Gaps |
|--------|-----------|--------------|------|
| W-latency | dt-obs-tracing/performance-analysis.md + request-attributes.md | ... | ... |
| W-cost    | dt-obs-tracing/request-attributes.md | ... | ... |
| W-loops   | dt-obs-tracing/request-attributes.md | ... | ... |
| W-econ    | dt-obs-tracing/request-attributes.md + dt-obs-services/service-metrics.md | ... | ... |

## Appendix B: Glossary
[Include ONLY if FULL_APPENDIX == true. Otherwise omit entirely.]
- gen_ai.* — OpenTelemetry semantic-convention attribute namespace for LLM
  spans (input tokens, output tokens, model, system, response ID).
- OpenLLMetry — the open-source library that emits OTel-conformant GenAI
  spans from common LLM SDKs.
- langgraph.node / crewai.task.id — framework-specific span attributes that
  identify which step of the graph or which agent task produced the span.
- Loop — a repeated identical (or near-identical) user prompt within a
  single session, indicative of an agent stuck in a reflexion / retry cycle.
- Cost-per-Request (CPR) — total LLM token cost for a feature divided by
  the number of root-span requests for that feature in the window.

## Appendix C: Pricing Table
[Include ONLY if FULL_APPENDIX == true. Otherwise: one line —
"Pricing table embedded in worker prompts; as of {pricing_table_used_as_of}.
Refresh from provider pricing pages before customer-shareable artifacts."]

## Appendix D: Query Exchange Log
[If FULL_APPENDIX == true: full DQL log with timing for all workers.
 If FULL_APPENDIX == false: one line —
 "Full query log: {LOG_PATH}". Do NOT inline the DQL.]

---

**Links:**
- [View Application](https://{TENANT}.apps.dynatrace.com/ui/entity/{APP_SERVICE_ID})
- [View Traces (window)](https://{TENANT}.apps.dynatrace.com/ui/apps/dynatrace.distributedtracing)

---

*End of Brief*
```

### Pre-Save Self-Check (run before writing anything)

If any item is "no", fix before writing:
- [ ] H1 title + `## Application ...` H2 subtitle present
- [ ] Header is `Generated / Analyst: Claude {MODEL} / Environment` (no
      tool-version banner)
- [ ] Executive Summary has all 4 sub-blocks (Headline, Window Totals,
      Top Concerns, Recommended Immediate Action)
- [ ] Latency Analysis has per-model + per-step + trend (3 subsections)
- [ ] Cost Top-N has per-model + top-25 + trend (3 subsections)
- [ ] Loop Detections has all 3 subsections + classification paragraph
- [ ] Per-Feature Economics has both rankings (total cost + CPR)
- [ ] Recommended Actions has all 3 lever subsections
- [ ] No DQL anywhere except Appendix D
- [ ] Appendix A present; B+C present only if `FULL_APPENDIX == true`; D
      present (full or log pointer)
- [ ] Footer is `**Links:**` + `*End of Brief*` (no tool-version signature)
- [ ] Filename will be `AIOBS_{APP_SLUG}_{DATE}[_SANITIZED]`
- [ ] **MD-only default:** the `.md` is written unconditionally; a `.pdf` is
      expected ONLY if `PDF_MODE == true` (and even then its absence is
      tolerated — `PDF engine unavailable` is non-fatal)
- [ ] **Pricing coverage:** every model carries a `Class`
      (priced / USD 0 self-hosted / UNPRICED); a `USD 0` or partially-unpriced
      total still rendered a full brief (RULE 10 — never abort on USD 0)
- [ ] **Mermaid lint** (same rules as dt-rcf — no colon in gantt task names,
      no `\n` in graph labels, no escaped `\"`, no `file:line` in labels)
- [ ] **Sanitization lint** (if CLEAN_MODE): no real APP name, no real
      session/conversation/trace IDs, no real tenant URL, no real customer
      feature names. Model names and framework names ARE preserved.

---

### Phase 5: Save Report (Markdown)

**Markdown is the canonical deliverable. Write the `.md` ALWAYS; generate a
PDF ONLY if `PDF_MODE == true`, and never let PDF failure abort the run.**

Filenames:
```
Normal:   AIOBS_{APP_SLUG}_{DATE}.md   (+ .pdf only if PDF_MODE)
Clean:    AIOBS_{APP_SLUG}_{DATE}_SANITIZED.md   (+ ...SANITIZED.pdf if PDF_MODE)

APP_SLUG examples:
  APP:"checkout-copilot"            → checkout-copilot
  APP:"support-agent"               → support-agent
  APP:"ai-travel-advisor-agent-test" → ai-travel-advisor-agent-test
  APP:"genai-demo" (model-bind)     → genai-demo
```

**Step 5.1 — Write the Markdown (always).** Generate the brief and write the
`.md`. This is the deliverable; the run is considered successful once it is
written.

**Step 5.2 — PDF (opt-in, best-effort).** ONLY if `PDF_MODE == true`:
- Borrow the dt-rca PDF mechanics (filename pairing, the intermediate
  `_pdf.md` Mermaid→ASCII conversion, the `md-to-pdf` invocation, cleanup) —
  read `~/.claude/skills/dt-rca/SKILL.md` for those mechanics ONLY. Do NOT
  adopt the dt-rca `Problem_*` filename; dt-ai-obs reports are `AIOBS_*`. Do
  NOT edit dt-rca.
- **Wrap PDF generation so a missing/broken engine is non-fatal:** if
  `md-to-pdf` (or the chosen engine) is unavailable or errors, log exactly
  ONE line — `PDF engine unavailable — Markdown only` — and CONTINUE. Never
  hard-fail a run on PDF.

If `PDF_MODE == false` (the default), skip Step 5.2 entirely and let Phase 6
report the PDF row as `skipped (MD-only default; pass --pdf to enable)`.

---

### Phase 6: Output Summary

```markdown
## dt-ai-obs Brief Generated

| Format | Filename |
|--------|----------|
| Markdown | AIOBS_{SLUG}.md |
| PDF | AIOBS_{SLUG}.pdf — OR — `skipped (MD-only default; pass --pdf to enable)` — OR — `PDF engine unavailable — Markdown only` |
| Log | AIOBS_{SLUG}.log |

**App:** {APP_DISPLAY_NAME} ({APP_SERVICE_ID})
**Framework:** {FRAMEWORK}
**Window:** {WINDOW.from} → {WINDOW.to}

### Headline Numbers
- **Total Spend:** ${TOTAL_COST_USD}
- **LLM Calls:** {TOTAL_CALLS}
- **Sessions:** {SESSION_COUNT}
- **Loops Detected:** {LOOP_COUNT_TOTAL}

### Top Concern
{rank_1_finding_one_liner}

### Worker Telemetry
| Worker | Findings | Gaps |
| W-latency | ... | ... |
| W-cost    | ... | ... |
| W-loops   | ... | ... |
| W-econ    | ... | ... |
```

If CLEAN_MODE, append sanitization key to console (never to file) per
dt-rca rules.

---

## QUALITY RULES (NON-NEGOTIABLE)

### Inherited from dt-rcf / dt-rca

R1–R10 below are dt-ai-obs-specific. The dt-rcf rules apply unchanged for
auth (workers never auth), single-message parallel dispatch, absence-gate
proof, Smartscape-native entity model, no backticks in DQL, single-quoted
dtctl args, `DTCTL_TOKEN_STORAGE=file` prefix on every dtctl call, and
de-duplication (each fact has ONE canonical home).

### New in dt-ai-obs

1. **Pricing table embedded in worker prompts.** W-cost and W-econ must
   carry the pricing table verbatim in their prompt — workers do not read
   it from disk. Every brief MUST state `pricing_table_used_as_of` in
   Appendix A. A stale price is a sensitive number; if Chris hasn't
   refreshed pricing in >30 days, the brief MUST be marked DRAFT.

2. **Framework hint is authoritative for non-`gen_ai.*` attributes.** The
   `--framework` flag drives which `langgraph.*` or `crewai.*` keys the
   workers query. If `gen_ai.system` disagrees with `--framework`, log a
   warning in Phase 0b and proceed — the framework flag wins for attribute
   selection because `gen_ai.system` is set per-call by the SDK.

3. **`gen_ai.*` are OpenTelemetry semantic conventions.** They are NOT
   Dynatrace-proprietary. The exact spelling matters — `gen_ai.usage.input_tokens`
   not `genai.input_tokens` or `llm.input_tokens`. Mark every inline DQL
   with `VALIDATE` on first production run and re-check against the current
   OTel GenAI spec quarterly.

4. **Loop detection has TWO axes — content-hash and step-frequency.** Never
   ship a brief that reports loops on only one. Content-hash catches
   identical-prompt cycles; step-frequency catches "same node firing N
   times" cycles where the prompt mutates each iteration. Both are real
   loops; both belong in the report.

5. **Cost-per-Request is required when a FEATURE_KEY resolves.** W-econ
   must compute it; the report must rank features by BOTH total cost AND
   CPR. A single ranking hides low-volume / high-margin-risk features.

6. **CLEAN_MODE sanitizes session / conversation / trace IDs.** These
   identify customers when combined with timestamp. Replace with
   `session-A1`, `session-A2`, etc. Preserve model names and framework
   names — they are not customer-identifying.

7. **DQL only in Appendix D** (when FULL_APPENDIX == true). Never in
   Recommended Actions or anywhere else in the body.

8. **Filename = `AIOBS_{APP_SLUG}_{DATE}[_SANITIZED]`** — NEVER `Problem_*`
   or `RCF_*`. The prefix identifies the artifact type for grep-ability.

9. **Window size guard.** Windows > 14 days risk exceeding orchestrator
   context when worker PhaseResults aggregate. Warn at parse time and
   suggest the user break the window or use `appendix=false` to compress
   the log.

10. **Model-pricing coverage — 3 classes, and `USD 0` NEVER aborts.**
    Every `gen_ai.request.model` resolves to exactly one class:
    - **priced** (commercial-API match) → token cost computed normally.
    - **self-hosted / compute-billed** (Ollama llama/mistral/orca-mini,
      genai-demo/genai-model) → token cost is a legitimate **USD 0**; the model
      and its token volume STILL appear in every table tagged `USD 0
      self-hosted (compute-billed)`. This is NOT an `unknown_pricing` gap.
    - **unpriced** (no match) → token cost excluded from `total_cost_usd`,
      `unknown_pricing: <model>` recorded in `gaps`, and the model + its
      token volume STILL surfaced in a dedicated "Unpriced Models" block so
      coverage is transparent. Never silently drop it; never zero it.

    **Hard guarantee:** a `total_cost_usd` of `USD 0` (all self-hosted) or a
    partially-unpriced total MUST still render a complete brief with the
    token-volume tables. The skill MUST NOT abort, emit an empty report, or
    treat `USD 0` as "no data". The old behavior (unknown → silent USD 0 → empty
    report) is the exact RULE-10 trap this fixes: on the live tenant the
    dominant model `genai-demo` plus the Ollama set are self-hosted, so a
    naive table produces `USD 0` for ~90% of calls. The dollar figure being
    small/zero is itself a finding (the spend is in compute), not a failure.

---

## PDF DIAGRAM RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"PDF DIAGRAM RULES". Summary: no Mermaid in PDFs — ASCII art only, inside
plain code blocks, 80-char max width.

---

## MERMAID SYNTAX RULES

Identical to dt-rcf. Read `~/.claude/skills/dt-rcf/SKILL.md` section
"MERMAID SYNTAX RULES". Summary: no colon in gantt task names; sequence
message labels have no literal double-quotes and no second colon; graph
labels use `<br/>` not `\n`.

---

## DIAGRAM SIZING RULES

Identical to dt-rca. Read `~/.claude/skills/dt-rca/SKILL.md` section
"DIAGRAM SIZING RULES".

---

## ERROR HANDLING

### APP not found (BOTH Phase 0b.0 probes empty)
This banner fires ONLY when Probe A (service) AND Probe B (model value) both
return zero rows — absence is gated, never assumed.
```markdown
## Error: APP Could Not Be Resolved

`APP: {APP_NAME}` matched neither a Dynatrace SERVICE (Probe A) nor a
`gen_ai.request.model` value (Probe B) in the window.

**Possible causes:**
- Name spelled differently in Dynatrace (use exact SERVICE name, or the
  exact model string from `gen_ai.request.model`)
- APP not registered as a Smartscape SERVICE node AND not a model value
- Wrong tenant — check `dtctl auth whoami`
- Window outside span retention

**Try:** list the real model values with
`fetch spans, from:now()-3d | filter isNotNull(gen_ai.request.model) |
 summarize count() by gen_ai.request.model | sort count desc`,
or partial-match a service via
`smartscapeNodes SERVICE | filter contains(name, "<partial>") | fields id, name`.
(`mcp__dynatrace__find_entity_by_name` is an optional accelerator when the
Dynatrace MCP is registered; it is unavailable on this tenant.)
```

### No LLM spans in window
```markdown
## Error: No LLM Spans Detected

No spans with `gen_ai.*` attributes were found for {APP_NAME} between
{WINDOW.from} and {WINDOW.to}.

**Possible causes:**
- OpenLLMetry instrumentation not deployed in this window
- LLM calls happening in a different service (check parent / child services)
- Window outside the span retention period

**Try:** Widen the window to 7d, or check
`fetch spans, from:now()-7d | filter isNotNull(gen_ai.request.model) |
 summarize count() by getNodeName(dt.smartscape.service) | sort count desc`
to find which services have LLM telemetry.
```

### Framework mismatch
Phase 0b warning is emitted, run continues. Report Appendix A documents the
mismatch ("`--framework langgraph` requested but `gen_ai.system=anthropic`
observed").

### Davis CoPilot unavailable
Generate brief from worker findings alone. Add banner in Davis Synthesis
section:
```
> **Note:** Davis CoPilot synthesis unavailable. Recommendations are
> derived from cross-worker findings only.
```

### Worker fails entirely
Orchestrator marks the corresponding section with `⚠️`, notes the gap in
Appendix A, and continues. Does not auto-retry beyond the single re-dispatch
described in Phase 1.

---

## BEGIN EXECUTION

Parse `$ARGUMENTS` and begin Phase 0a. Execute all phases autonomously until
the brief is saved.

**Do not stop for confirmation between phases. Do not ask questions. Generate
the complete brief.**
