---
name: setup-python
description: "Read when adding Risicare to a service for the first time: install, API key, `init()`, first traced call, flush."
---
# Setup (Python)

## Prerequisites

- CPython 3.10 or later.
- The user has a project API key from the Risicare dashboard. Sign-up is by invitation only.
- You output the detection summary.

## Steps

1. Read the installed version from the lockfile, `python -m pip show risicare`, or (no pip in
   the venv) `python -c "import importlib.metadata as m; print(m.version('risicare'))"`. A
   `PackageNotFoundError` means "not installed". If it is older than 0.6.0: Stop. Return to
   the SKILL.md router.
2. Add `risicare>=0.6.0` to the project's dependency file with its own package manager.
   Example: `pip install 'risicare>=0.6.0'`. It needs `risicare-core>=0.1.8`; do not pin
   `risicare-core` lower. Quote every version range and every extra: `pip install 'risicare[langchain]'`.
3. Do not add an extra for a package that the project already depends on. An extra only pulls
   the third-party package. Extras do not exist for `openai`, `anthropic`, `cohere`,
   `google-generativeai` and `mistralai`.
4. Tell the user to set `RISICARE_API_KEY` in the environment or a secret store. The SDK reads
   the process environment only; it does not load `.env`. For a `.env` file, load it with
   `python-dotenv` (`load_dotenv()`) before `risicare.init()`. Check the key without printing it:
   `python -c "import os; print('set' if os.environ.get('RISICARE_API_KEY','').strip() else 'missing')"`.
5. Set no endpoint. Remove `RISICARE_ENDPOINT=https://app.risicare.ai` if it is present.
6. In the entry point, `import risicare` and call `risicare.init()` once. The import order of
   provider and framework packages does not matter.
7. Group each unit of work in `@risicare.trace` or `with risicare.trace(name=...)`. An LLM call
   outside a trace becomes its own trace.
8. In a short-lived process, call `risicare.flush()` once before exit.
9. Run the Verify loop.

## Code

<!-- checked against risicare 0.6.0 -->
```python
import openai
import risicare

risicare.init()                   # reads RISICARE_API_KEY; set nothing else
client = openai.OpenAI()
with risicare.trace(name="first-call"):
    client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": "Hi"}])
    trace_id = risicare.get_current_trace_id()
delivered = risicare.flush()      # short-lived process: deliver before exit
```

`init()` takes `api_key` by position and every other argument by keyword. A call with two or
more positional arguments raises `TypeError`. Pass only what the user asks for, for example
`environment="production"` or `service_name="checkout"`. The default `environment` is
`"development"`.

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A script, a CLI, a job or a serverless handler | `risicare.flush()` before exit | `os._exit`, SIGKILL and an OOM kill skip the exit drain |
| `multiprocessing` fork, gunicorn `--preload`, celery prefork | `risicare.flush()` at the end of the child's work | A forked child has no SIGTERM flush and can exit with `os._exit` |
| `multiprocessing` spawn or forkserver | `risicare.init()` inside the child | A spawned child has no SDK state |
| `multiprocessing.Pool` or `ProcessPoolExecutor` | `risicare.flush()` at the end of each task. With spawn (the default on macOS and Windows), also `initializer=risicare.init` | A `Pool` stops its workers with SIGTERM, and a fork worker has no exit drain. A spawn worker without `init()` sends nothing |
| A request handler or an agent loop | `@risicare.trace` around the unit of work | Groups its LLM calls into one trace |
| Named agents, sessions, phases | Stop. Return to the SKILL.md router | Tiers 2 to 5 |
| `RISICARE_TRACING=true`, `RISICARE_API_KEY` set, and no `init()` call | Keep it: this is tier 0, auto-init on `import risicare` | Same defaults as `init()`: `environment="development"`, no `service.name` |
| `RISICARE_TRACING=true` and an `init(service_name=..., environment=...)` call | Unset the variable | `init()` with arguments replaces the import-time client. Spans made before `init()` keep the defaults |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| No `RISICARE_API_KEY` | One WARNING `no API key is set. The SDK sends nothing to Risicare`; `is_enabled()` and `flush()` are `False` | Tell the user to set the key. The WARNING says `pass api_key="rsk-..."`: do not. The environment variable wins; never write the key in code |
| Key only in a `.env` file | The same WARNING: the SDK does not read `.env` | Load it with `python-dotenv` before `init()` |
| A second `init()` with other arguments | The new arguments are ignored without an error | Keep one `init()` at process start |
| `RISICARE_TRACING` set to `off`, `false`, `0`, `no` or an unknown value | Tracing is off. An unknown value logs one WARNING `is not a recognised value, so tracing is OFF` | Unset it, or set `true` |
| `debug=True` | Whole span payloads printed to stdout, unredacted | Remove it |
| `exporters=[...]` passed to `init()` | WARNING `Custom exporters provided — automatic HttpExporter disabled`; nothing reaches Risicare | Remove it |
| `flush()` returned `True`, so "spans arrived" | `True` means every accepted span was delivered. It is also `True` when the code made no span | Require exported spans ≥ EXPECTED, and a trace id that is not `None` |
| `init(key, endpoint, project_id)` written for 0.3.0 | `TypeError: init() takes from 0 to 1 positional arguments` | Pass keywords |

## Verification

Run the Verify loop. Call `risicare.flush()` one time at the end and trust its boolean. When
it is `False`, read `risicare.get_metrics()`: `exported_spans` (accepted spans only),
`rejected_spans`, `failed_scores`, `queue_size`, `dropped_spans` and `dropped_spans_by_reason`
(`queue_full`, `transport_refused`, `shutdown_residue`, `unacknowledged`). `failed_exports` is
not a loss counter. `risicare.get_current_trace_id()` inside the trace gives the id; it is `None`
when tracing is off.
