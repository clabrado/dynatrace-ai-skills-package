---
name: dt-obs-log-parser
description: >-
  Log pattern detection, DQL parse statement generation, and OpenPipeline processing rule
  deployment. Use when extracting structured fields from raw log content, identifying log
  formats across services, or automating field extraction at ingestion time.
  Trigger: "parse logs", "extract fields from logs", "log pattern detection", "create parse
  rule", "log parsing", "OpenPipeline log processor", "structured log extraction",
  "log field extraction", "generate parse statement", "parse rule", "identify log format".
  Do NOT use for general log searching or filtering (use dt-obs-logs), explaining existing
  DQL queries, or Dynatrace product documentation questions.
license: Apache-2.0
---

# Log Parser Skill

Analyze live Dynatrace log data to detect structural patterns, generate validated DQL `parse` expressions, and optionally deploy OpenPipeline processors that extract fields at ingestion time.

## What This Skill Covers

- Sampling representative log lines from a service or across the environment
- Detecting structural patterns (JSON, Log4j, key-value, syslog, Nginx, custom)
- Generating and validating DQL `parse` expressions for each detected format
- Deploying OpenPipeline processing rules to extract fields at ingestion time

---

## Input

`$ARGUMENTS` may include any combination of:
- A service or process group name (e.g., `payment-service`, `checkoutservice`)
- A time range (e.g., `2h`, `24h`) — defaults to `1h` if omitted
- A severity filter (e.g., `ERROR`, `WARN`)
- Nothing — perform a broad analysis across all available logs

---

## Phase 1: Sample Collection

**Goal:** Collect a diverse, representative set of raw log lines.

1. Build a DQL query to fetch ~300 log entries from the specified scope. Do not filter by status unless the user requested it — variety across severity levels improves pattern coverage.

   If a service or process was specified, add a process group filter:
   ```dql
   fetch logs, from:now() - 1h
   | filter contains(dt.process_group.detected_name, "service-name")
   | fields content, status, process_group = dt.process_group.detected_name
   | limit 300
   ```

   Otherwise, sample broadly:
   ```dql
   fetch logs, from:now() - 1h
   | fields content, status, process_group = dt.process_group.detected_name
   | limit 300
   ```

2. Execute the query using the DQL execution tool.

3. If no results are returned: inform the user, suggest widening the time range or verifying log ingestion, and stop.

4. Present a brief summary before proceeding:
   - Total records sampled
   - Distinct process groups found
   - Distribution of severity levels (ERROR / WARN / INFO / etc.)

---

## Phase 2: Pattern Detection

Analyze the `content` field across all sampled records. Identify **distinct structural patterns** by inspecting the raw text.

### Detection checklist

| Pattern type | Key indicators |
|---|---|
| **JSON** | Content starts with `{` or `[` |
| **Log4j / Logback** | `YYYY-MM-DD HH:mm:ss.SSS LEVEL [thread] logger - message` |
| **Key=value pairs** | `key=value key2=value2` or `key="quoted value"` |
| **Syslog (RFC 5424/3164)** | `<priority> TIMESTAMP HOSTNAME APP-NAME: message` |
| **Nginx / Apache access** | `IP - - [timestamp] "METHOD /path HTTP/x.x" status bytes` |
| **Simple structured** | `TIMESTAMP LEVEL message` (space-delimited fields) |
| **Custom / unstructured** | Recurring delimiters, tokens, or fixed-width columns |

For each distinct pattern found:
- Show 2–3 representative example log lines
- Describe the structure in plain language
- Note which fields can be extracted
- Count how many of the 300 sampled records match this pattern

Present this as a numbered list and **pause for the user to confirm** which patterns to generate parse rules for before continuing.

---

## Phase 3: DQL Parse Statement Generation

For each confirmed pattern, generate a DQL `parse` expression using the primitives below.

### DQL parse primitive reference

| Primitive | Matches |
|---|---|
| `TIMESTAMP('format'):name` | Timestamp with explicit format string |
| `TIMESTAMP:name` | Auto-detected timestamp |
| `IPADDR:name` | IPv4 or IPv6 address |
| `INT:name` | Integer number |
| `FLOAT:name` | Floating-point number |
| `WORD:name` | Non-whitespace token |
| `STRING:name` | Quoted string (strips surrounding quotes) |
| `DATA:name` | Arbitrary data up to the next pattern element |
| `LD:name` | Line data — captures the rest of the line |
| `EOL` | End of line |
| `JSON:name` | JSON object or array |
| `'literal'` | Exact literal text to match and skip |
| `SPACE` | One or more whitespace characters |
| `(pattern)?` | Optional group |
| `KVP{sep='=' pair_sep=' '}:name` | Key-value pairs (configurable separators) |

### Output format per pattern

```dql
// Pattern N: <description>
// Extracted fields: field1, field2, field3
// Estimated coverage: ~<count> of <total> sampled records

fetch logs, from:now() - 1h
| filter <pre-filter expression that identifies this pattern>
| parse content, "<pattern expression>"
| fields timestamp, status, field1, field2, field3
| limit 50
```

### Validation step (required)

After generating each expression, execute it against a limit-5 sample:

```dql
fetch logs, from:now() - 1h
| filter <pre-filter>
| parse content, "<pattern>"
| fields timestamp, field1, field2, field3
| limit 5
```

- If all extracted fields return non-null values: mark the expression as **validated**.
- If any fields are null/empty: show the failing raw lines, revise the expression, and re-test. Repeat until validated or explicitly note that the pattern could not be reliably parsed.

Present all validated parse statements together before moving to Phase 4.

---

## Phase 4: OpenPipeline Rule Generation (optional — requires explicit confirmation)

After presenting the validated DQL parse statements, ask the user:

> "Would you like to convert any of these patterns into OpenPipeline processing rules? OpenPipeline extracts fields at **ingestion time**, making them available as first-class attributes across all queries, dashboards, and alerts — without needing `parse` in every DQL statement.
>
> **Note: this will modify your live Dynatrace environment configuration.** Which patterns (if any) would you like to deploy?"

If the user selects one or more patterns, proceed with the following steps.

### Step 1 — Check environment variables

Verify `DT_TENANT_URL` and `DT_API_TOKEN` are available:

```bash
echo "Tenant: ${DT_TENANT_URL:-NOT SET}"
echo "Token: ${DT_API_TOKEN:+SET}${DT_API_TOKEN:-NOT SET}"
```

If either is missing, ask the user to provide them before continuing.

### Step 2 — Fetch current OpenPipeline logs configuration

```bash
curl -s -X GET "${DT_TENANT_URL}/platform/classic/environment-api/v2/openpipeline/v1/configurations/logs" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" \
  -H "Accept: application/json"
```

If this call fails (non-2xx response), show the full error and stop — do not attempt a PUT.

### Step 3 — Generate processor JSON

For each selected pattern, create a processor entry to add to the default pipeline's `processing.processors` array:

```json
{
  "id": "custom-parser-<descriptive-slug>",
  "type": "dql",
  "enabled": true,
  "description": "Extract fields from <pattern description> logs",
  "matcher": "<a matchesPhrase() or contains() expression that reliably identifies this log format>",
  "dqlFunction": "parse(content, \"<validated pattern expression>\")"
}
```

**Matcher guidance:**
- Use a distinctive literal token from the format (e.g., a fixed prefix, a unique keyword)
- Prefer `matchesPhrase()` for phrase matching, `contains()` for simple substrings
- Make it specific enough to avoid matching unrelated log lines

### Step 4 — Show diff and require confirmation

Present clearly what will change:

```
Pipeline:   default (or the relevant pipeline name)
Action:     ADD processor

ID:          custom-parser-<slug>
Description: <description>
Matcher:     <matcher expression>
Parser:      parse(content, "<pattern>")
Extracted:   field1, field2, field3

Estimated coverage: ~<N> logs/hour based on sample rate
```

Then ask explicitly:

> **"Deploy this processor to OpenPipeline? Reply 'yes' to confirm or 'no' to skip."**

Do not proceed until the user explicitly replies "yes".

### Step 5 — Deploy

Merge the new processor(s) into the existing configuration JSON and PUT it back:

```bash
curl -s -X PUT "${DT_TENANT_URL}/platform/classic/environment-api/v2/openpipeline/v1/configurations/logs" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '<updated config JSON>'
```

- **HTTP 200/204**: Confirm success to the user and show the processor ID.
- **Any error**: Show the full response body, do not retry automatically, and ask the user how to proceed.

---

## Error Handling

| Situation | Response |
|---|---|
| No logs returned in Phase 1 | Inform user, suggest wider time range or checking OneAgent log ingestion |
| Parse expression returns all nulls | Show 3 raw failing lines, revise the expression, re-test before presenting |
| `DT_TENANT_URL` / `DT_API_TOKEN` not set | Ask user to set them; do not attempt API calls |
| OpenPipeline GET returns error | Show full error, stop — do not attempt PUT |
| OpenPipeline PUT returns error | Show full response, do not retry, ask user how to proceed |

---

## Related Skills

- **dt-obs-logs** — Ad-hoc log querying and filtering
- **dt-dql-essentials** — Full DQL syntax and operator reference
- **dt-obs-tracing** — Correlate extracted log fields with distributed traces
