---
name: scores-javascript
description: "Read when the user wants to record a quality score on a trace or report a caught exception."
---
# Scores and errors (JavaScript)

## Prerequisites

- Setup is done, with `RISICARE_API_KEY` set. `score()` sends nothing without a key, or when
  tracing is off.
- The api host limits scores to 3,000 requests per 5 min per IP address.
- The code has a point where a quality value is known, or a `catch` block that handles an error
  and continues.

## Steps

1. Read the trace id inside a span: `span.traceId` in a `startSpan` callback, or
   `getCurrentTraceId()` inside that span. Outside a span it is `undefined`, also inside
   `withSession`. Never take it from `getTraceContext()` at the top level or in a plain
   function: the next span does not use that id. Never make up an id.
2. Call `score(traceId, name, value, { spanId, comment })`. `value` is a finite number in
   [0, 1]. The options object is optional. Python passes the options as keywords instead.
3. `score()` returns `void`, not a Promise. Do not `await` it and do not read a result.
4. In a `catch` block that does not re-throw, call `reportError(err)`.
5. `score()` stops its request after 2000 ms and follows no redirect. A status of 300 or more,
   an error or a timeout writes a stderr WARNING and adds 1 to `getMetrics().failedScores`.
   Later failures within 10 s share one WARNING `N more scores failed after the last warning`.
6. `await flush()` waits for pending scores. It returns `false` after a failed score. When it
   returns `true`, say "accepted by the api". Never say that the score is in the dashboard.

## Code

<!-- checked against risicare 0.8.0 (ESM run against a loopback sink; tsc strict + verbatimModuleSyntax) -->
```ts
import { getTracer, score, reportError, flush, getMetrics } from 'risicare';
const tracer = getTracer(); // undefined before init(): then the work runs untraced
const answer = tracer
  ? await tracer.startSpan({ name: 'answer-question' }, async (span) => {
      const a = await runAgent(question);
      score(span.traceId, 'factual_accuracy', 0.9, { comment: 'checked against source' });
      return a;
    }) : await runAgent(question);
try { await fetchDocs(); } catch (err) { reportError(err); }
if (!(await flush())) console.warn('risicare: not delivered', getMetrics().failedScores);
```

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| An evaluator, a grader or user feedback with a value | `score(traceId, name, value)` | Sends the value for the trace |
| A value on another scale (1–5, 0–100) | Scale it into [0, 1] first | A value outside the range, NaN or a non-number writes one WARNING and is not sent. `failedScores` and `flush()` do not show it |
| A score that belongs to one step | `{ spanId }` of that span | Ties the score to the span |
| Free text with the score | `{ comment }` | It passes through `mask` under the key `score.comment`. It is sent also with content capture off: KNOWN-ISSUE (risicare-sdk #68) |
| An error that is caught and not re-thrown | `reportError(err)` | Makes an error span with an `error.code`; the same error makes at most one span per 5 min |
| An error thrown inside `startSpan` | Nothing | The span records it and sets `error.code` |

`reportError(error, { name, attributes })` makes an error span `error:<ClassName>` unless you
pass `name`. It never throws, and it does nothing before `init()`. An error that no rule
matches gets `TOOL.EXECUTION.CRASHED`.

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `await score(...)` or `.then()` on it | It returns `void`; `.then()` throws `TypeError` | Call it and continue |
| A made-up trace id | Not 32 hex: HTTP 422 and a stderr WARNING `score "<name>" failed: HTTP 422`. 32 hex: accepted, but it attaches to no trace | Use `span.traceId` inside the span |
| `process.exit()` right after `score()` | The score is lost | `await flush()` before the exit, or let the process end on its own |
| `endpoint` set to the user's own host and `RISICARE_API_ENDPOINT` to another | Scores go to the `endpoint` host: the option comes first | Pass `apiEndpoint` to `init()`, or set no endpoint |
| Telling the user "the score is recorded" | Not provable from the SDK | Say "accepted by the api" only when `flush()` returned `true` |

## Verification

Run the Verify loop. `await flush()` covers the scores. When it returns `false` and
`failedScores` is above 0, read the stderr WARNING `score "<name>" failed:` for the HTTP status
or the error. A value that the SDK refused shows only as a stderr WARNING `score: the value`.
