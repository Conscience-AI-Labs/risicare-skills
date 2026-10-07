---
name: troubleshooting
description: "Read when traces do not appear, a `risicare` WARNING appears, or a call is not traced."
---
# Troubleshooting

Find the symptom. Do the check before the fix. Never print the API key while you check. Each
export WARNING names the HTTP status or the error class. For more detail in one run: Python
`logging.getLogger("risicare").setLevel(logging.DEBUG)` (with a handler, for example
`logging.basicConfig(level=logging.WARNING)`); JavaScript `RISICARE_DEBUG=true` with the key
set. Never raise the root logger to DEBUG: a provider SDK's own logger can then print prompt
text. Never use `debug=True` for this. Remove the change after the run.

## No spans at all

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| Python WARNING `no API key is set. The SDK sends nothing to Risicare`; `is_enabled()` and `flush()` are false | No API key in the process (also a key of only spaces) | Is `RISICARE_API_KEY` in the process environment? The SDK does not load `.env` | User sets the key. For `.env`: `python-dotenv`, or `node --env-file=.env` |
| JavaScript stderr WARNING `Risicare found no API key`; trace id is 32 zeros | No API key: tracing is off without one, also with `RISICARE_TRACING=true` | `isEnabled()` returns `false` | As above |
| Nothing traced | `RISICARE_TRACING` is `false`, `0`, `no` or `off` (no message), or an unknown value (WARNING `is not a recognised value, so tracing is OFF`) | Read the variable | Unset it, or set `true` |
| New `init()` options have no effect | A second `init()`: the first one wins. JavaScript warns only when `apiKey`, `endpoint`, `environment`, `serviceName`, `serviceVersion` or `sampleRate` differ | Search for every `init(` call | Keep one `init()`, at process start |
| Python WARNING `Custom exporters provided — automatic HttpExporter disabled` | `init(exporters=[...])` replaced the HTTP exporter | Search for `exporters=` | Remove it |
| Python WARNING `HttpExporter(s) with no API key were left out` (`add_exporter()`: `an HttpExporter with no API key was not added`) | An `HttpExporter` that you built has no `api_key=`, also one that points at your own relay. Nothing is sent | Search for `HttpExporter(` | Pass `api_key=` to it, or remove it |
| Spans of the last seconds are missing | The process exits before the queue drains (`os._exit`, SIGKILL, `process.exit()`, serverless freeze). JavaScript: an uncaught exception or rejection sends no spans: KNOWN-ISSUE (risicare-sdk #66) | How does the process end? | Python `risicare.flush()`; JavaScript `await flush()` before exit, also in a top-level `catch` |
| Python child or pool-worker spans missing | A forked child and a pool worker have no SIGTERM flush; a fork child can exit with `os._exit`. A spawn child or worker without `init()` sends nothing (spawn is the default on macOS and Windows) | `multiprocessing`, `Pool`, `ProcessPoolExecutor`, gunicorn `--preload`, celery prefork | `risicare.flush()` at the end of each task; spawn children call `init()`, spawn pools take `initializer=risicare.init` |

## WARNING lines from the SDK

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| `spans are NOT reaching <host>` with HTTP 307, an HTML 200, or "not a Risicare ack" | `RISICARE_ENDPOINT` or `endpoint` points at a host that is not the gateway | Read the variable and the `init()` call | Remove it; set no endpoint |
| `score … failed: HTTP 307`, and `RISICARE_ENDPOINT=https://app.risicare.ai` is set | Spans pass through a temporary route; scores go to the dashboard host and fail. After the route is removed, spans fail too | Read the variable | Remove it |
| `spans are NOT reaching <host>`: HTTP 401 or 403, a server error, or `cannot be reached` | Refused key, a server error, or no answer | The WARNING names the status or the error | 401 or 403: user checks the key in the dashboard. Otherwise check network egress to `ingest.risicare.ai` |
| `span(s) DISCARDED (reason=transport_refused)` | The same failures, after 4 export rounds. JavaScript says "up to 12 HTTP requests"; a 401 sends 4: KNOWN-ISSUE (risicare-sdk #57). The text can name the key after a 500: KNOWN-ISSUE (risicare-sdk #56) | Read the first WARNING for the status | As above |
| `the backend rejected N of M span(s)` | The gateway refused some spans; they are lost | Read the detail at the end of the line | Report it to the user with the detail |
| `the backend accepted 0 of N span(s)`, or `acknowledged 0 of N` | The gateway refused or did not count the whole batch | Read the detail | Report it to the user with the detail |
| `refused this batch for capacity (HTTP 429/503)` | Rate limit or load shedding. The batch is retried | Is it constant? Does `flush()` return false? | Rare: no action. Constant: lower the export rate or batch size |
| `the API key does not look like a Risicare key` | The key has no known prefix. Tracing continues with this key | Is the variable set from the project key in the dashboard? Do not print it | Tell the user to copy the full key from the dashboard again. Do not edit the key |
| `Span queue full` | More than 10,000 spans wait in memory | Span rate against export rate | Lower the span count, or fix the export failure first |
| Python `shutdown() stopped with …`; JavaScript `shutdown() gave up with …`: "not confirmed as delivered" | The exit drain ran out of time | Container stop grace period | Give the process more time to stop; `shutdown(timeout)` sets the bound |
| `a span ended after shutdown()` (Python: `or the SDK's drain`), `reason=shutdown_residue` | A span ended after the SDK stopped taking spans, for example in your own SIGTERM handler. It is not sent. One `flush()` is false | `get_metrics()` / `getMetrics()`: `shutdown_residue` | End the span before the signal. JavaScript: register the listener before `init()` |
| Python `Fix load failed`; JavaScript `Fix loader circuit breaker opened`; both `FixRuntime initialization failed` | FixRuntime is on, or the SDK is older than PyPI 0.4.0 / npm 0.7.0 | `fix_runtime`, `fixRuntime`, `RISICARE_FIX_RUNTIME`, the version | Turn it off; upgrade an old SDK |

## A call is not traced, or traced wrongly

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| Python AutoGen: `attached-inert` WARNING | Bare `import autogen_agentchat` | The import line | `from autogen_agentchat.agents import ...` |
| Python `cohere` 7.x: an export WARNING with HTTP 400 (`expected u32`); the whole batch is lost | The v1 client sends token counts as floats (known defect) | `cohere` version | Tell the user; the other spans of that batch are lost too |
| Python: every OpenTelemetry span end raises `AttributeError` | `init(otel_bridge=True)` with `opentelemetry-sdk` 1.45 and an SDK `TracerProvider` (known defect). Without a `TracerProvider`, nothing is bridged | `otel_bridge=` | Remove it; tell the user |
| Python: LlamaIndex, Instructor or Pydantic AI send two spans for one call | Known defects of 0.6.0 | — | Tell the user |
| JavaScript LangGraph `graph.invoke()`: each LangChain span two times (tokens double) | Known defect; `graph.stream()` sends each one time | — | Tell the user. Do not add a `RisicareCallbackHandler` too |
| JavaScript: two LLM spans per call | `patchOpenAI` applied two times: KNOWN-ISSUE (risicare-sdk #67); two `RisicareCallbackHandler` instances; `patchInstructor` over a patched client | Search for each patch and handler | Patch each client one time; pass one handler. Instructor: keep `patchOpenAI`; remove `patchInstructor`, or accept the second span |
| JavaScript: no LLM span | The original client or graph is called, not the returned object | Search for the variable | Use what `patchX(...)` or `instrumentLangGraph(...)` returns |
| JavaScript: `does not provide an export named 'Span'` (or `'Tracer'`) | Value import of a type | The import line | `import type { Span, Tracer } from 'risicare'` |
| JavaScript: `does not provide an export named 'patchGoogle'` | Wrong name from old docs | The import line | `patchGoogleAI` |
| JavaScript: the callback never runs | `session()`, `agent()` or `traceThink()` returns a wrapper; or `getTracer()?.startSpan(...)` ran before `init()` or after `shutdown()` | Is the result called? Is `init()` first? | Call the wrapper, or use `withSession` / `withAgent` / `withPhase`; guard the tracer so the work always runs |
| Each LLM call is its own trace | No enclosing trace | Is the call inside `trace` / `startSpan`? | Wrap the unit of work |
| Python: later, unrelated spans are children of an ended span, in one trace | A generator that holds an open Risicare block (trace, session, agent, phase) was not read to its end: KNOWN-ISSUE (risicare-sdk #64) | `async for ... break` over such a generator | Read it to its end inside the span, or close it there (`contextlib.aclosing`) |
| Python: a stream call has no LLM span | The provider stream was not read to its end and not closed: KNOWN-ISSUE (risicare-sdk #65) | `break` in the stream loop | Read it to its end, or call `stream.close()` |
| No prompt text in spans, or `<risicare:content-omitted>` | SDK switch off (default): provider spans have no text attribute. SDK switch on and project setting off: the gateway writes the placeholder | `trace_content` / `traceContent`, project setting | Both on, only if the user asks. JavaScript: only `patchOpenAI` and `patchAnthropic` record text |

## Scores, error codes and cost

| Symptom | Cause | Check | Fix |
|---|---|---|---|
| A score does not appear | WARNING `score … failed: <reason>`: an endpoint setting, a non-hex trace id (422), or a short `flush(timeout)`: KNOWN-ISSUE (risicare-sdk #55). WARNING `score: the value must be in [0.0, 1.0]`: not sent, counted in failed scores, so the next `flush()` is false. No WARNING: no key, tracing off, or JavaScript `process.exit()` right after `score()`. Over 3,000 requests per 5 min per IP | `flush()` result, failed scores, the WARNING lines | Set no endpoint; fix the value; use the default `flush()` timeout; `await flush()` before exit |
| Many errors read `TOOL.EXECUTION.CRASHED`, severity 5 | No SDK rule matches the error; this is the fallback code | — | Tell the user that the code is coarse |
| Dashboard cost of cached traffic differs from the provider bill | Cached tokens are not priced as cached: the SDKs send no cached-token count. OpenAI-style input counts include them at the full input rate: too high. Anthropic, Bedrock and Vertex Claude input counts leave them out: too low. A fix is planned, with no date | — | Tell the user |
| Cost marked "estimated" | The server table does not know the model id: Bedrock (`us.anthropic.…`, ARNs), Vertex (`…@…`), `provider/model` (LiteLLM, OpenRouter), some new models | The span's model name | Tell the user that it is the fallback price |
| A call has no cost | No LLM span or no token counts, for example the OpenAI Responses API: KNOWN-ISSUE (risicare-sdk #78). Python: OpenAI `with_raw_response` calls, also from CrewAI | Does its LLM span have token counts? | Tell the user |
