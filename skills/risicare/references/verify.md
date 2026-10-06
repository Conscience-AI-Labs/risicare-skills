---
name: verify
description: "Read after every change, to prove that spans arrive (required)."
---
# Verify

The task is not done until this loop passes. It needs only the ingest host, which is open to
every source address (an egress filter can still block it). Never poll an API for the trace.

## Steps

1. Check that `RISICARE_API_KEY` is in the process environment, without printing it. The SDK
   does not load `.env`. No key: stop, and tell the user to create one in the dashboard. With
   no key, `flush()` returns true and sends nothing. Python `is_enabled()` stays true.
2. Turn debug off (`debug=True`, `debug: true`, `RISICARE_DEBUG`). It falsifies the result.
3. Set EXPECTED. Count one span per trace or `startSpan`, traced LLM call and agent span
   (`@agent`, `agent()` wrapper call). Count one per phase or multi-agent span, `reportError`
   (Python: `report_error` outside a span) and `tracedStream`. Count one `session.end` per
   decorated session call or per `session_context` / `withSession` block, not per session.
   `withAgent`, `withPhase`, `agent_context`, `phase_context` add none. One trace and one LLM call: 2.
4. Frameworks add a span per chain, node and event. JavaScript LangGraph `invoke()` sends each
   LangChain span two times. Then the ≥ check proves only that "a span arrives", not "traced".
5. Run the path once (the entry point, or a temporary script that you delete after). Capture
   warnings in a file. Python: the `risicare` logger (stderr, also under app logging).
   JavaScript: BOTH stderr (`[risicare] WARNING:` lines) AND `console.warn` (init warnings).
6. After `init()`, require `is_enabled()` / `isEnabled()`. False: `RISICARE_TRACING` is an off
   value, or (JavaScript) no key. JavaScript with `RISICARE_TRACING=true` and no key: true, and
   `flush()` false. Python: require no `exporters=` in `init()` (it replaces the HTTP exporter).
7. Call `flush()` one time at the end and trust a false. Require true, and exported spans ≥ EXPECTED.
   It is false after any span dropped, rejected or unacknowledged, or any failed score, since the
   previous `flush()` returned (since `init()` for the first call). When it is false, read the
   metrics: dropped spans by reason, rejected spans, failed scores. False with no loss counted: the
   deadline ended with a batch in flight. In a signal handler or after `shutdown()` it can be true
   with no span delivered: call `shutdown()`, then require exported ≥ EXPECTED, 0 dropped, 0 rejected.
8. Do not require failed exports to be 0. A batch that got 429 for one retry round and was
   then delivered counts 1, and `flush()` is true.
9. `flush()` returns by its deadline (default 5000 ms). It only waits; it never stops a request.
   To end a process in a fixed time, call `shutdown(timeout)`: Python
   `risicare.shutdown(timeout_ms=2000)`, JavaScript `await shutdown(2000)`.
10. Print the trace id. Give the user the dashboard link. On a failure, fix the cause and run
    the loop again. Stop. Return to the SKILL.md router.

## Code

<!-- Python, the run. Checked against risicare 0.5.1: run as written against a loopback sink -->
```python
import risicare
EXPECTED = 2                          # one trace + one LLM call (step 3)
risicare.init()
assert risicare.is_enabled(), "tracing is off: no key, or RISICARE_TRACING"
with risicare.trace(name="verify"):
    run_instrumented_path()           # the code you changed
    trace_id = risicare.get_current_trace_id()
```
<!-- Python, the check. Checked against risicare 0.5.1: passes on `ok`; fails on 401, partial-reject, 429, 503 -->
```python
ok = risicare.flush()                 # one time, at the end; not in a signal handler
m = risicare.get_metrics()
assert ok and trace_id and m["exported_spans"] >= EXPECTED, m
print("risicare trace id:", trace_id)
```
<!-- JavaScript, the run. Checked against risicare 0.8.0: run as written; tsc strict + verbatimModuleSyntax -->
```ts
import { init, flush, getMetrics, getTracer, isEnabled } from 'risicare';
const EXPECTED = 2; // one span + one LLM call (step 3)
init(); // once; patch the clients as in setup
const tracer = getTracer();
if (!tracer || !isEnabled()) throw new Error('risicare: tracing is off (no key, or RISICARE_TRACING)');
let traceId = '';
await tracer.startSpan({ name: 'verify' }, async (span) => { traceId = span.traceId; await runInstrumentedPath(); });
```
<!-- JavaScript, the check. Checked against risicare 0.8.0: passes on `ok`; fails on 401, partial-reject, 429, 503 -->
```ts
const ok = await flush(); // one time, at the end; not in a signal handler
const m = getMetrics();
if (!ok || m.exportedSpans < EXPECTED) throw new Error(`risicare: not delivered ${JSON.stringify(m)}`);
console.log('risicare trace id:', traceId);
```

A failed check ends the process. JavaScript ends at once. Python runs its exit drain first, at
most about 6 s more on a stalled server.

## The dashboard link

- Ask the user for the project UUID: it is in the dashboard URL, not in the key or the SDK.
  Link: `https://app.risicare.ai/project/<project-uuid>/traces/<trace-id>`. The user must be
  signed in and a member of the project's organization.
- Without the UUID: `https://app.risicare.ai/traces/<trace-id>`. It opens the user's current
  project, which may not be the key's project; say so. Visible within seconds, later under load.

## What the loop proves

Proves (with no Python `exporters=`): the gateway took the key and queued at least EXPECTED
spans of this run. No span was dropped, rejected or left unacknowledged, and no score failed.
Not proven: storage, visibility, the key's project (the user checks the dashboard), or that
each LLM call was traced when a framework adds spans. A score value outside [0, 1] is refused in the SDK with one
WARNING; `flush()` does not show it.

Report what you instrumented (files, tier), the SDK version, the `flush()` result and metrics,
and the trace id and link. Also report what is not done or not verified (integrations, scores).
