---
name: tiers
description: "Read when choosing how much instrumentation to add (tiers 0 to 5)."
---
# Instrumentation tiers

Each tier adds to the one below it. Start at the lowest tier that meets the request. Propose a
higher tier only when the code shows the need, and say why in the detection summary.

| Tier | Name | What it adds | When the code shows |
|---|---|---|---|
| 0 | Zero code | Auto-init on `import risicare` from two environment variables. Python only | The user wants traces with no code change |
| 1 | Explicit init | One `init()` call, traced LLM calls, and a trace around each unit of work | Any service. This is the default |
| 2 | Agent identity | An agent name, role and id on spans | Functions or classes that act as agents |
| 3 | Sessions | A session id and user id across several traces, and a session end marker | A conversation, chat or user id across turns |
| 4 | Decision phases | Think, decide, act and observe spans | An agent loop with planning, choice and tool steps |
| 5 | Multi-agent | Message, delegation and coordination spans between agents | Agents that send work or messages to other agents |

## Tier 0

- Python only. JavaScript has no import-time init; it starts at tier 1.
- Set `RISICARE_TRACING` to an on value and `RISICARE_API_KEY`. `import risicare` then starts
  the SDK. A later `init(...)` with arguments replaces the import-time client; spans made
  before it keep the defaults.
- Its defaults are those of `init()`: `environment` is `development`, with no service name.
- It gives no trace grouping: each LLM call is its own trace. Prefer tier 1 when you edit code.

## Tier 1

- One `init()` at process start. In Python, provider packages are traced automatically, in
  any import order; in JavaScript, each client is wrapped explicitly.
- Wrap each unit of work (a request, a job, an agent run) in a trace, so its LLM calls group
  into one trace. Without it, every LLM call is its own trace.
- Flush before exit in short-lived processes. In a Python pool worker, flush at the end of
  each task. With the spawn start method, also call `init()` in the worker.

## Tiers 2 and 3

- An agent is a named actor with a role. Use one of the SDK's `AgentRole` values; the SDK does
  not check it.
- A session groups the traces of one conversation. Use an id that already exists in the code:
  a conversation id, chat id or thread id. Add the user id when the code has one.
- A session emits one `session.end` marker span per decorated call or per session block
  (`session_context`, `withSession`), not one per session.

## Tiers 4 and 5

- Phases label steps of an agent loop: think (reason, plan), decide (choose), act (run a tool),
  observe (read a result). A phase can also be set as context only, with no span.
- Multi-agent spans record a message to an agent, a delegation to an agent, or the coordination
  of several agents.
- Add these only to the agent loop's real steps. A span on every helper adds noise.

## Across all tiers

- Scores attach a value in [0, 1] to a trace. A failed score logs a WARNING and makes
  `flush()` return false.
- Reported errors record a caught exception with a coarse error code from the Risicare
  taxonomy. An error that no SDK rule matches reads `TOOL.EXECUTION.CRASHED`.
  An exception that leaves a traced function is recorded without extra code.
- Prompt and completion text are off by default at every tier.
