---
name: architecture
description: "Read when you need the hosts, what a customer can reach or the ingest answer. Also read for the content and PII rules, or for when a trace becomes visible."
---
# Architecture

## Hosts

| Host | What the SDK sends there | Customer with a project key |
|---|---|---|
| `ingest.risicare.ai` | Spans: `POST /v1/spans`. The gateway also takes `/v1/spans/batch`, `/v1/traces` and `/v1/otlp/v1/traces` | Yes |
| `api.risicare.ai` | Scores: `POST /v1/scores`. Limit: 3,000 requests per 5 min per IP address | Yes, for this path. Every other path that a customer needs answers 403, including `GET /v1/fixes/active`, `/v1/projects` and `/openapi.json` |
| `app.risicare.ai` | Nothing, in the current SDK. It is the dashboard | Signed-in people only. `/api/v1/*` answers 401 to an API key |

The SDK defaults are `https://ingest.risicare.ai` (spans) and `https://api.risicare.ai` (scores).
Set no endpoint. Scores go to the first of: the `api_endpoint` / `apiEndpoint` argument; the
`endpoint` argument; `RISICARE_API_ENDPOINT`; `RISICARE_ENDPOINT` when no `endpoint` argument is
given; `https://api.risicare.ai`. The two `endpoint` settings are skipped when they name the
default ingest host. So a leftover `RISICARE_ENDPOINT=https://app.risicare.ai` sends scores to
the dashboard. It answers 307 to its login page. The SDK follows no redirect, and logs
`score … failed: HTTP 307`. Remove it.

`app.risicare.ai` still takes `POST /v1/spans` through a temporary route, so spans sent there
arrive today. Its removal is scheduled; the date is not fixed. After the removal, spans sent
there fail, with an export WARNING. Do not send new code there.

## What a customer can and cannot do

- Write spans and scores. Nothing else by API.
- Read nothing by API. The dashboard is the only read path. No trace read-back, MCP server,
  CLI or trace delete exists.
- FixRuntime polls `GET /v1/fixes/active`. That route is closed at the edge and empty in the
  beta. Fix application is held. Never enable FixRuntime.
- Projects, API keys, team, alerts and webhook settings are in the dashboard only.

## Authentication

- The gateway reads only `Authorization: Bearer <key>`. A request with only `X-API-Key` gets
  401. The SDK sends Bearer on every call.
- The key is opaque. The dashboard issues it; the user pastes it exactly. Do not check a
  prefix, a length or a format. Do not print or log it.

## The ingest answer

| Answer | Meaning |
|---|---|
| 200 `{accepted, rejected}`, plus `errors: [{index, error}]` and `ignored` only when they are not empty | The accepted spans are in the queue. A 200 does not mean they are stored yet |
| 200 with `rejected > 0` and `accepted > 0` | The rejected spans are lost. They are not retried |
| 200 with `rejected > 0` and `accepted: 0` | All spans are lost. The SDK counts them in rejected spans and does not retry |
| 200 `{accepted: 0, rejected: 0}` | The SDK counts the batch as not delivered and retries it; then it drops it (`transport_refused`) |
| 401 | Missing or refused key |
| 429 `RATE_LIMIT_EXCEEDED` | Gateway rate limit, by default: 10,000 requests/s per project, 100 requests/s per IP. Has `Retry-After`. The edge's 429 has none |
| 503 `TOO_MANY_SPANS`, `TOO_MANY_ATTRIBUTES` | Over the per-request cap: 10,000 spans or 24,000 attribute pairs. Permanent for that payload: a retry fails again |
| 413 `PAYLOAD_TOO_LARGE`, `DECOMPRESSED_PAYLOAD_TOO_LARGE` | Request body over 10 MB, or a gzip body that inflates past it |
| 503 `QUEUE_NEAR_CAPACITY`, `REDIS_MEMORY_HIGH`, `BUFFER_FULL`, `ENQUEUE_UNCONFIRMED` | The spans were not queued (load shedding or a queue failure); retry later |

The SDK reads this answer for you. It retries 429 and 503 and writes a WARNING when spans are
rejected or dropped. Not every 503 is retryable (see the caps). Among the 503 codes, only
`REDIS_MEMORY_HIGH` and `BUFFER_FULL` send `Retry-After`. The metrics give the counts: exported
spans (accepted only), rejected spans, and the drop reason `unacknowledged` for spans that the
answer does not account for.

## Content and PII

- Prompt and completion text reach storage only when BOTH switches are on. One is the SDK's
  `trace_content` / `traceContent` (off by default). The other is the project's content setting.
- With the SDK switch off, provider spans carry no prompt or completion attribute. A content
  attribute that the code sets by hand becomes `<risicare:content-omitted>` before export.
- With the SDK switch on and the project setting off, the gateway replaces the text with
  `<risicare:content-omitted>`.
- Provider error text is NOT under these switches. `status_message`, `exceptions[].message`
  and `exceptions[].stacktrace` are kept with content off, by design: error diagnosis reads
  them. That text can repeat the prompt. The server redacts PII in it. Tell the user.
- `mask=` / `mask` is an optional callback that edits span fields before export.
- Not content, so sent with content off: tool-call names, the tool count and the finish
  reason. Keep user data out of tool names. A score comment is sent too: KNOWN-ISSUE (risicare-sdk #68).
- The SDK does no PII redaction. Risicare removes PII on the server.
- Turn content capture on only when the user asks. Then tell the user that the project's
  content setting must be on too.

## Cost

- The server calculates the cost of each LLM span from its price table. For a model that the
  table knows, the server price wins over a cost that the span carries (the JavaScript SDK
  sends its own). For an unknown model, a sent cost is kept; with none, the fallback price
  applies and the dashboard marks the cost "estimated".
- Cached tokens are not priced as cached: the SDKs send no cached-token count. Where the input
  count includes cached tokens (OpenAI style), they cost the full input rate: too high.
  Anthropic, Bedrock and Vertex Claude input counts leave them out, so they cost nothing: too low.
- Bedrock, Vertex and `provider/model` ids get the fallback price. Batch discounts and priority,
  regional or long-context tiers are not applied.
- A call with no LLM span or no token counts has no cost. Never present the dashboard cost as
  the price or the bill.

## Visibility

- A span is usually visible in the dashboard within seconds of the 200. It takes longer when the
  server is under back-pressure; the spans wait in the queue.
- Visibility cannot be checked by API. Give the user the dashboard link and let them look.
