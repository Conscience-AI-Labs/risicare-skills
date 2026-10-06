---
name: phases-multiagent-javascript
description: "Read when the code has think, decide and act steps (tier 4). Also read when agents message, delegate to or coordinate other agents (tier 5)."
---
# Phases and multi-agent (JavaScript)

## Prerequisites

- Setup is done and the Verify loop passed once.
- Agents are already identified (tier 2), if the code has more than one agent.

## Steps

1. Map the agent loop to phases. Reasoning and planning → think. Choosing an option or a tool
   → decide. Running a tool or an action → act. Reading a result → observe.
2. Wrap each step function with its phase wrapper: `traceThink`, `traceDecide`, `traceAct` or
   `traceObserve`. Each returns a wrapper; each call of the wrapper is one span.
3. To set a phase without a span, run the block in `withPhase(SemanticPhase.X, fn)`. It runs
   now.
4. For agent-to-agent traffic, wrap the function that sends or hands off: `traceMessage`,
   `traceDelegate` or `traceCoordinate`. These also return wrappers.
5. Run the Verify loop. Add one expected span per wrapper call.

## Code

<!-- checked against risicare 0.8.0 (ESM run against a loopback sink; tsc strict + verbatimModuleSyntax) -->
```ts
import { traceThink, traceAct, traceDelegate, withPhase, SemanticPhase } from 'risicare';

const analyse = traceThink('analyse', async (task: string) => planSteps(task));
const search = traceAct('call-search', async (q: string) => runSearch(q));
const handOff = traceDelegate({ to: 'agent-writer' }, async (draft: string) => writer(draft));

const steps = await analyse(task); // call the wrapper
await withPhase(SemanticPhase.DECIDE, () => choose(steps)); // runs now, no span
```

The code is TypeScript. For plain JavaScript (`.js`, `.mjs`), remove the type annotations
(`: string`).

Span kinds: think → `think`, decide → `decide`, act → `tool_call`, observe → `observe`,
message → `message`, delegate → `delegation`, coordinate → `coordination`. A name that you
pass is the span name as given. Without a name, a phase span is `<phase>:<function>` (for
example `think:analyseQuery`), as in Python; an anonymous function gives `phase:think`. Always
pass a name as the first argument. `to` is stored as `message.target_agent_id`: pass the target
agent's `agentId`.

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A planning or reasoning step | `traceThink('name', fn)` | Phase THINK |
| A choice between tools, routes or answers | `traceDecide('name', fn)` | Phase DECIDE |
| A tool call or side effect | `traceAct('name', fn)` | Phase ACT, kind `tool_call` |
| Code that reads a tool result | `traceObserve('name', fn)` | Phase OBSERVE |
| A phase that must not create a span | `withPhase(SemanticPhase.THINK, fn)` | Context only; pass the enum |
| One agent sends a message to another | `traceMessage({ to: agentId, type }, fn)` | Span `message:<type>→<to>`; `type` defaults to `MessageType.REQUEST` |
| One agent hands a task to another | `traceDelegate({ to: agentId }, fn)` | Span `delegate→<to>` |
| One agent runs a group of agents | `traceCoordinate({ participants: ['a', 'b'] }, fn)` | Span `coordinate:[a,b]` |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `await traceThink('x', fn)` with no trailing call | `fn` never runs; no span | `await traceThink('x', fn)()`, or keep the wrapper and call it |
| `traceDelegate('writer', fn)` (also `traceMessage`, `traceCoordinate`) | No error; a span `delegate→undefined` (`message:request→undefined`); `traceCoordinate` throws `TypeError` | Pass an options object: `traceDelegate({ to: 'writer' }, fn)` |
| Phase wrappers on every small helper | Noise in the trace | Wrap only the agent loop's steps |

## Verification

Run the Verify loop. Count one span per wrapper call in the expected number.
`withPhase` and `withAgent` add no span.
