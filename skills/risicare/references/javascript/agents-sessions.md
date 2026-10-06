---
name: agents-sessions-javascript
description: "Read when the code has named agents or multi-turn conversations: agent identity (tier 2) and sessions (tier 3)."
---
# Agents and sessions (JavaScript)

## Prerequisites

- Setup is done and the Verify loop passed once.
- The code has a function or class that acts as an agent. Or it has a conversation id, chat id
  or user id that spans several calls.

## Steps

1. Choose the form. `withAgent` and `withSession` run the callback now. `agent()` and
   `session()` return a wrapper and run nothing until you call the wrapper. Python has
   decorators and context managers instead; do not carry that habit across.
2. To trace one agent as a span, wrap its function once at module scope with
   `agent({ name, role }, fn)` and call the wrapper. It opens an agent span `agent:<name>`.
   Always pass `name`. Without it, the agent takes the function's name, and an anonymous
   function gives `agent:agent`, one shared identity.
3. To set agent identity without a span, use `withAgent({ agentId, name, role }, fn)`. Always
   pass `name`: without it the name is `agent`. Pass `agentId`: without it each call is a new
   agent.
4. For a conversation turn, run the handler in `withSession({ sessionId, userId }, fn)`. Open a
   span inside it (guarded, as in setup), so the turn is one trace and has a trace id. Without
   the span, each call is its own trace and `getCurrentTraceId()` returns `undefined`.
5. Run the Verify loop. Each `withSession` call (or `session()` wrapper call) adds one
   `session.end` marker span, also for the same session id: count each one.

## Code

<!-- checked against risicare 0.8.0 (ESM run against a loopback sink; tsc strict + verbatimModuleSyntax) -->
```ts
import { agent, withAgent, withSession, getTracer } from 'risicare';
const plan = agent({ name: 'planner', role: 'planner' }, async (task: string) => runPlanner(task));
const turn = async () => {
  await withAgent({ agentId: 'agent-1', name: 'researcher', role: 'worker' }, () => research());
  await plan('draft an answer'); // call the wrapper
};
const tracer = getTracer(); // the guard runs the turn also when tracing is off
await withSession({ sessionId: conversationId, userId }, () => (tracer ? tracer.startSpan({ name: 'turn' }, turn) : turn()));
```

The code is TypeScript. For plain JavaScript (`.js`, `.mjs`), remove the type annotations
(`: string`).

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A function that runs one agent | `const run = agent({ name, role }, fn)` at module scope, then `run(...)` | One agent span per call; the id is fixed when you wrap |
| An agent id from config or a database | `withAgent({ agentId, name, role }, fn)` | Context only, no span. Without `agentId` each call gets a new id `<name>-<16 hex>` |
| A conversation or chat id in a handler | `withSession({ sessionId, userId }, fn)` | Runs now; emits one `session.end` marker per call |
| A handler that receives the id as an argument | `session((id: string) => ({ sessionId: id }), fn)` | The resolver derives the options from the call arguments |
| `role` values | `orchestrator`, `worker`, `supervisor`, `specialist`, `router`, `aggregator`, `broadcaster`, `critic`, `planner`, `executor`, `retriever`, `validator`, `reviewer`, `custom` | `AgentRole` members; `coordinator` is not one |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `await session({ sessionId }, () => work())` | The callback never runs; no error | `withSession({ sessionId }, () => work())` |
| `agent()` with no `name` | The function's name becomes the agent name; an anonymous function gives `agent:agent` | Always pass `name` |
| `withAgent` with no `name` | All such agents have the name `agent` | Always pass `name` |
| `agent(...)` called per request | A new agent id per request | Wrap once at module scope |
| `agentRole` in the options | Ignored; the option is `role` | Use `role` |
| Expecting `metadata`, `version`, `turnNumber` or `parentSessionId` on spans | Stored in context only | Do not rely on them in the dashboard |
| Expecting `serviceName` on agent spans | `service.name` is set only on spans outside an agent | Use the agent name to filter |

## Verification

Run the Verify loop. Expect one more span per `agent()` wrapper call and one `session.end`
marker span per `withSession` call (two turns of one session give two). `withAgent` adds no span.
