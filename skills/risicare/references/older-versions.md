---
name: older-versions
description: "Read when the installed SDK is older than PyPI 0.5.1 or npm 0.8.0, or when the user asks to upgrade the SDK."
---
# Older versions and upgrades

## Read the installed version

Use package metadata, never a constant in the source.

- Python: the lockfile (`uv.lock`, `poetry.lock`, `Pipfile.lock`) or `python -m pip show risicare`.
  In a `uv` venv with no pip: `uv pip show risicare` or
  `python -c "import importlib.metadata as m; print(m.version('risicare'))"`.
- JavaScript: the lockfile, or `npm ls risicare --depth=0`. Last resort: read
  `node_modules/risicare/package.json` as a file. Never
  `require('risicare/package.json')`: the package does not export it.

## Bands

| Band | Python (PyPI `risicare`) | JavaScript (npm `risicare`) | What to do |
|---|---|---|---|
| A, current | 0.5.1 and later | 0.8.x and later | Teach the current API. Set no endpoint |
| B | 0.4.0 | 0.7.0 | Give the Band B message, then upgrade |
| C | 0.2.2 and 0.3.0 | 0.5.2 and 0.6.0 | Give the Band B and C messages. Do not teach the old API |
| D | 0.2.0 and earlier | 0.5.0 and earlier | Give the Band B, C and D messages |

PyPI has no 0.2.1 and npm has no 0.5.1.

Python 0.5.0: teach the current API, then upgrade. 0.5.1 fixes these 0.5.0 defects. On 0.5.0, Pydantic AI `run_stream()` raises `TypeError`, and a classic `AgentExecutor` run or a LlamaIndex run is split into many traces. 0.5.0 does not export `extend_span_ttl`.

## Upgrade messages

Say these to the user, in your own words, before you change code.

Band B (and every older band):
- "On this version, `flush()` can return true when nothing was delivered, and a failed score
  gives no message."
- Python: "A traced `stream=True` call of OpenAI, Anthropic, Groq, Cerebras or Together, and a
  LangChain `ChatOpenAI` stream, can raise `TypeError`. A provider package imported after
  `init()` can stay untraced. A fork pool can hang at its end."
- JavaScript: "A value import of `Span` or `Tracer` passes `tsc` and fails at load. Patched
  clients lose `withResponse()`; Anthropic `messages.stream()` and Hugging Face
  `chatCompletion` fail."

Band C, add:
- "Your SDK version sends spans to `app.risicare.ai` through a temporary route. The removal of
  that route is scheduled; the date is not fixed. After the removal, this version cannot
  deliver spans."
- "Scores from this version do not reach Risicare. It uses the old `/api` paths."
- "This version starts FixRuntime whenever a key is set. Its fix requests fail." Python logs a
  `Fix load failed` WARNING per attempt. JavaScript logs at debug level, and prints a WARNING
  only when the fix loader's circuit breaker opens.
- "This version is MIT. The upgrade changes `risicare` and `risicare-core` to a proprietary
  licence, and makes FixRuntime opt-in." A new install of an old `risicare` also takes
  `risicare-core` 0.1.8, which is proprietary; `risicare-core<0.1.8` keeps the MIT core.
- JavaScript 0.6.0 and earlier: "This version follows redirects. A redirect to an HTML 200
  page counts as delivered." Python 0.3.0 fails loudly on a redirect (`spans are NOT reaching`).

Band D, add: "This version treats any 2xx answer as delivered. It can report success for spans
that were lost." Python: "Upgrade `risicare-core` with it." Python 0.2.0 has no `flush()`, so
the Band B `flush()` message does not apply to it.

If the user asked you to update or upgrade, do it. In your report, tell the user that the licence
changes from MIT to proprietary (Bands C and D). If the user did not ask, ask before you upgrade:
the licence change is the user's decision. Remove a dashboard endpoint in both cases.

## What can break on the upgrade

- Python `init()`: every argument after `api_key` is keyword-only. A positional call raises `TypeError`.
- `flush()` returns false after any loss. Code that reads its result now sees the losses.
- `exported_spans` / `exportedSpans` counts accepted spans only. New: `rejected_spans`,
  `failed_scores` and the drop reason `unacknowledged` (camelCase in JavaScript).
- Stored names: Python tier 0 `environment` is `development` (was `production`), with no
  service name. JavaScript: an unnamed `agent()` takes the function name; a phase span is
  `<phase>:<function>`. Saved dashboard filters on old names stop matching.
- JavaScript: `Span` and `Tracer` are types only. `new Span(...)` and `instanceof Tracer` do
  not compile. Use `import type`.
- `RISICARE_TRACING`: an empty value is now unset, so tracing runs when a key is set.
- Python pool workers have no SIGTERM flush. Add `risicare.flush()` at the end of each task.

## Upgrade steps

1. Python: `pip install -U 'risicare>=0.5.1'` (it needs `risicare-core>=0.1.8`). JavaScript:
   `npm install 'risicare@^0.8.0'`. Use the project's package manager.
2. Fix each item of "What can break" that the code uses.
3. Remove `RISICARE_ENDPOINT=https://app.risicare.ai` and any `endpoint` option with that host.
4. A custom or proxy `RISICARE_ENDPOINT` stays. Scores and fix reads then go to that host at
   `/v1/...`, without `/api`. A failure logs `failed: HTTP <status>` and counts in failed
   scores. Ask the user to make the proxy serve `/v1/scores`, or to set the api endpoint.
   An `endpoint` argument beats `RISICARE_API_ENDPOINT`: pass `api_endpoint=` / `apiEndpoint`.
5. Remove any `/api/v1/...` URL that the code builds by hand. The SDK composes its own URLs.
6. Remove any FixRuntime setting. Do not turn FixRuntime on.
7. Run the Verify loop.

## Band A with a leftover endpoint

`RISICARE_ENDPOINT=https://app.risicare.ai` on a Band A SDK sends spans through the temporary
route and scores to the dashboard host. Scores fail now: WARNING `score '<name>' failed:
HTTP 307` (JavaScript `score "<name>"`). Spans fail when the route is removed. Remove the
variable. Do not replace it with another host.
