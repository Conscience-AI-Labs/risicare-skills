---
name: scores-python
description: "Read when the user wants to record a quality score on a trace or report a caught exception."
---
# Scores and errors (Python)

## Prerequisites

- Setup is done, with `RISICARE_API_KEY` set. `score()` sends nothing without a key.
- The api host limits scores to 3,000 requests per 5 min per IP address.
- The code has a point where a quality value is known, or an `except` block that handles an
  error and continues.

## Steps

1. Inside the trace, read the trace id: `trace_id = risicare.get_current_trace_id()`. It is
   32 lowercase hex characters. Never make up an id such as `"trace-abc123"`: the api refuses
   it (HTTP 422).
2. Call `risicare.score(trace_id, name, value)`. `value` is a float in [0.0, 1.0]. The options
   `span_id=` and `comment=` are keyword-only.
3. In an `except` block that does not re-raise, call `risicare.report_error(exc)`.
4. Call `risicare.flush()` one time before the process ends. It waits for the score requests
   and returns `False` when a score failed. A normal exit also waits, at most 2.5 s.
5. When `flush()` returned `True`, tell the user that the api accepted the score. Never say
   that it shows in the dashboard.

## Code

<!-- checked against risicare 0.5.1 -->
```python
with risicare.trace(name="answer-question"):
    answer = run_agent(question)
    trace_id = risicare.get_current_trace_id()
risicare.score(trace_id, "factual_accuracy", 0.9, comment="checked against source")
try:
    fetch_docs()
except TimeoutError as exc:
    risicare.report_error(exc)
if not risicare.flush():                    # waits for the score request
    print(risicare.get_metrics()["failed_scores"])
```

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| An evaluator, a grader or user feedback with a value | `risicare.score(trace_id, name, value)` | Sends the value for the trace |
| A value on another scale (1–5, 0–100) | Scale it into [0.0, 1.0] first | A value outside the range is not sent (one WARNING). `failed_scores` and `flush()` do not show it |
| A score that belongs to one step | `span_id=` of that span | Ties the score to the span |
| Free text with the score | `comment=` | It passes through `mask=` under the key `score.comment`. It is sent also with content capture off: KNOWN-ISSUE (risicare-sdk #68) |
| An exception that is caught and not re-raised | `risicare.report_error(exc)` | Records it with a coarse `error.code`. Outside a span, the same error makes at most one span per 5 min; inside a span, every call is recorded |
| An exception that propagates out of a traced function | Nothing | The trace span records it and sets `error.code` |

An error that the SDK cannot classify gets `TOOL.EXECUTION.CRASHED`, severity 5. A
`ValueError` and a `KeyError` get it; a `TimeoutError` gets `TOOL.EXECUTION.TIMEOUT`.

`report_error(exc, *, name=None, attributes=None)` records on the current span. Outside a span
it makes a separate span `error:{ExceptionType}`. It never raises.

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| A made-up trace id | Not 32 hex: HTTP 422 and a WARNING `score '<name>' failed`. 32 hex: accepted, but it attaches to no trace | Use `get_current_trace_id()` inside the trace |
| An empty or `None` trace id | A WARNING `trace_id must be a non-empty string`; not sent | As above |
| `RISICARE_API_ENDPOINT`, `api_endpoint=`, `RISICARE_ENDPOINT` alone, or `endpoint=` set to a host that is not the default ingest host | Scores go to that host. A redirect is not followed. Each failure logs `score '<name>' failed: HTTP <status>` | Set no endpoint |
| Reading `score()`'s return value | It returns `None` | Read `flush()`, then `get_metrics()["failed_scores"]` |
| Telling the user "the score is recorded" | The SDK proves only that the api accepted it | Say "accepted by the api" |

## Verification

Call `risicare.flush()` one time at the end. When it is `False`, read
`risicare.get_metrics()["failed_scores"]` and the WARNING `score '<name>' failed: <reason>`.
Later failures within 10 s share one WARNING: `N more scores failed after the last warning`.
Run the Verify loop for the spans.
