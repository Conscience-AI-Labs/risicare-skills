---
name: phases-multiagent-python
description: "Read when the code has think, decide and act steps (tier 4). Also read when agents message, delegate to or coordinate other agents (tier 5)."
---
# Phases and multi-agent (Python)

## Prerequisites

- Setup is done and the Verify loop passed once.
- Agents are already identified (tier 2), if the code has more than one agent.

## Steps

1. Map the agent loop to phases. Reasoning and planning → think. Choosing an option or a tool
   → decide. Running a tool or an action → act. Reading a result → observe.
2. Decorate each step function with its phase decorator: `@risicare.trace_think`,
   `@risicare.trace_decide`, `@risicare.trace_act` or `@risicare.trace_observe`. Each call
   becomes one span with that phase.
3. For a block inside a function, use `with risicare.trace_think("name"):` (also
   `trace_decide`, `trace_act`). `trace_observe` has no `with` form.
4. For agent-to-agent traffic, decorate the function that sends or hands off:
   `@risicare.trace_message`, `@risicare.trace_delegate` or `@risicare.trace_coordinate`.
5. Run the Verify loop. Add one expected span per decorated call.

## Code

<!-- checked against risicare 0.5.1; `target=` form proven by probe, not by a test -->
```python
@risicare.trace_think
def analyse(task): ...
@risicare.trace_decide("choose-tool")
def choose(analysis): ...

with risicare.trace_act("call-search"):
    results = search(query)

@risicare.trace_delegate(target="agent-writer", target_name="writer")
def hand_off(draft): ...
```

Span kinds: think → `think`, decide → `decide`, act → `tool_call`, observe → `observe`,
message → `message`, delegate → `delegation`, coordinate → `coordination`. The default span
name of a decorator is `{phase}:{function name}`, for example `think:analyse`. Message and
delegate use their phase: `communicate:notify`, `coordinate:hand_off`. A name that you pass
replaces the default: `@risicare.trace_decide("choose-tool")` gives `choose-tool`.

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A planning or reasoning step | `@risicare.trace_think` | Phase THINK |
| A choice between tools, routes or answers | `@risicare.trace_decide` | Phase DECIDE |
| A tool call or side effect | `@risicare.trace_act` | Phase ACT, kind `tool_call` |
| Code that reads a tool result | `@risicare.trace_observe` | Phase OBSERVE; decorator only |
| A phase that must not create a span | `with risicare.phase_context(SemanticPhase.THINK):` | Context only; pass the enum, not a string |
| One agent sends a message to another | `@risicare.trace_message(target=..., target_name=...)` | Kind `message`, phase COMMUNICATE |
| One agent hands a task to another | `@risicare.trace_delegate(target=..., target_name=...)` | Kind `delegation`, phase COORDINATE |
| One agent runs a group of agents | `@risicare.trace_coordinate` | Kind `coordination`, phase COORDINATE |

`SemanticPhase` is exported by `risicare`: `from risicare import SemanticPhase`.

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `with risicare.trace_observe(...)` or `@risicare.trace_observe("name")` | `TypeError` or `AttributeError`: decorator only, no positional name | Use bare `@risicare.trace_observe` or `name=` |
| `with risicare.trace_message(...)` (also delegate, coordinate) | `TypeError`: decorator only | Decorate the sending function |
| `phase_context("think")` | `AttributeError: 'str' object has no attribute 'value'` | `phase_context(SemanticPhase.THINK)` |
| Phase decorators on every small helper | Noise in the trace | Decorate only the agent loop's steps |

## Verification

Run the Verify loop. Count one span per decorated call in the expected number.
`phase_context` and `agent_context` add no span.
