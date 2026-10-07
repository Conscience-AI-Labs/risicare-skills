---
name: setup-javascript
description: "Read when adding Risicare to a service for the first time: install, API key, `init()`, first traced call, flush."
---
# Setup (JavaScript)

## Prerequisites

- Node 18 or later. Node 18 is the stated floor and is not tested: KNOWN-ISSUE (risicare-sdk #60).
  The SDK is for the Node runtime only; no edge build exists.
- The user has a project API key from the Risicare dashboard. Sign-up is by invitation only.
- You output the detection summary.

## Steps

1. Read the installed version with `npm ls risicare --depth=0` (exit 1 and `(empty)` means "not
   installed") or from the lockfile. Never `require('risicare/package.json')`: the package does
   not export it. If the version is older than 0.9.0: Stop. Return to the SKILL.md router.
2. Add `risicare` 0.9.0 or later with the project's package manager (`^0.9.0` in
   `package.json`). Example: `npm install 'risicare@^0.9.0'`.
3. Tell the user to set `RISICARE_API_KEY` in the environment or a secret store. The SDK reads
   the process environment only; it does not load `.env`. For a `.env` file, start Node with
   `node --env-file=.env`. Check the key without printing it:
   `node -e "console.log(process.env.RISICARE_API_KEY ? 'set' : 'missing')"`.
4. Set no endpoint. Remove `RISICARE_ENDPOINT=https://app.risicare.ai` if it is present.
5. In the entry point, call `init()` once, before any other Risicare call. It is synchronous,
   returns nothing and reads the key from the environment.
6. Nothing is traced automatically. Wrap each LLM client with its `patchX` function and use the
   returned object. OpenAI is below. For other clients: Stop. Return to the SKILL.md router.
7. Group each unit of work in a span, with a guard that always runs the work (code below).
   Never write `getTracer()?.startSpan(...)`: before `init()` or after `shutdown()` it skips
   the callback, and so the user's own work. An LLM call outside a span is its own trace.
8. In a short-lived process, `await flush()` before exit. It waits at most 5000 ms by default
   (`flush(timeoutMs)`). It only waits: it never stops a request that is already sent.
9. Run the Verify loop.

## Code

<!-- checked against risicare 0.9.0 (ESM run against a loopback sink; tsc strict + verbatimModuleSyntax) -->
```ts
import OpenAI from 'openai';
import { init, flush, getTracer } from 'risicare';
import { patchOpenAI } from 'risicare/openai';
init(); // reads RISICARE_API_KEY; set nothing else
const openai = patchOpenAI(new OpenAI()); // use the returned proxy
const run = () => openai.chat.completions.create({ model: 'gpt-4o', messages: [{ role: 'user', content: 'Hi' }] });
const tracer = getTracer(); // undefined before init() and after shutdown()
const reply = tracer ? await tracer.startSpan({ name: 'first-call' }, run) : await run();
await flush(); // short-lived process: deliver before exit
```

ESM with top-level `await` is shown. CommonJS works too: `const { init } = require('risicare')`.
Pass only the `init()` options that the user asks for, for example `environment: 'production'`
or `serviceName: 'checkout'`. The default `environment` is `'development'`. Take the trace id
inside a span (`span.traceId` in the callback). Do not take it from `getTraceContext()` outside
a `withSession`, `withAgent` or `withPhase` callback: the next span does not use that id.

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A serverless handler (Lambda, a Vercel Node function) | `await flush()` before the handler returns | Deliver before the handler returns. `flush()` returns within its timeout (default 5000 ms) |
| `process.exit()`, or a crash path | `await flush()` or `await shutdown()` before it | `process.exit()` skips the exit drain. An uncaught exception or rejection sends no spans: KNOWN-ISSUE (risicare-sdk #66) |
| A process that must stop in a fixed time | `await shutdown(timeoutMs)` | Only `shutdown()` stops an export. After `flush()` returned `false`, a stalled export kept Node alive for up to about 20 s |
| Its own `SIGTERM` handler | Register it before `init()` if it makes spans that must be delivered. In it, `await flush()`, then read `getMetrics()` | The listener runs once in both orders, and its exit code stands. `flush()` in it waits for the SDK's own drain, and `true` means delivered. A span that the listener ends after the drain is `shutdown_residue`: one `flush()` is `false`. An early `process.exit()` cuts the drain |
| Next.js, Cloudflare Workers, Vercel Edge, Deno Deploy | Ask the user; Node runtime only | No test or doc covers these runtimes |
| A request handler or an agent loop | The guarded span from the code above around the unit of work | Groups its LLM calls into one trace |
| Named agents, sessions, phases | Stop. Return to the SKILL.md router | Tiers 2 to 5. JavaScript has no tier 0 |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| No `RISICARE_API_KEY` (also: key only in `.env`) | Nothing is sent; one stderr WARNING `Risicare found no API key`; `isEnabled()` and `flush()` are `false`; trace id is 32 zeros | Set the key; `node --env-file=.env`. The WARNING says `Pass apiKey to init()`: do not. The environment variable wins; never write the key in code |
| Calls on the original client | Not traced | Use the object that `patchOpenAI(...)` returns |
| `import risicare from 'risicare'` | `SyntaxError` at load (ESM); `tsc` error TS1192 | Use named imports, or `import * as risicare` |
| `import { Span, Tracer } from 'risicare'` | `tsc` error TS1484 with `verbatimModuleSyntax`. Plain JavaScript: `SyntaxError` at load (ESM), `undefined` (CommonJS) | Always `import type { Span, Tracer } from 'risicare'` |
| `flush()` or `shutdown()` without `await`, then `process.exit()` or a serverless return | The last spans are lost | `await` both |
| A short `flush(timeoutMs)` to end the process fast | `false`, and a stalled export keeps Node alive | `await shutdown(timeoutMs)` |
| `RISICARE_TRACING` set to a value other than `true 1 yes on false 0 no off` | Tracing off; one stderr WARNING names the value | Unset it, or set `true` |
| A second `init()` with other options | The first call wins. A `console.warn` only when `apiKey`, `endpoint`, `environment`, `serviceName`, `serviceVersion` or `sampleRate` differ; other options are ignored silently | Keep one `init()` at process start |
| `exporters: [...]` passed to `init()` | Ignored without a message | Remove it |

## Verification

Run the Verify loop. Call `await flush()` one time at the end and trust its boolean. When it is
`false`, read `getMetrics()`: `droppedSpansByReason`, `rejectedSpans`, `failedScores`. None raised
means that the deadline ended with a batch in flight. `failedExports` is not a loss counter.
The trace id is `span.traceId` in the `startSpan` callback, or `getCurrentTraceId()` inside that
span. WARNINGs (`[risicare] WARNING:`) go to `process.stderr.write`. The init conflict warning
goes to `console.warn`. Capture both.
