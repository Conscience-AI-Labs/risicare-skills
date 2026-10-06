---
name: agents-sessions-python
description: "Read when the code has named agents or multi-turn conversations: agent identity (tier 2) and sessions (tier 3)."
---
# Agents and sessions (Python)

## Prerequisites

- Setup is done and the Verify loop passed once.
- The code has a function or class that acts as an agent. Or it has a conversation id, chat id
  or user id that spans several calls.

## Steps

1. Find the function that runs one agent. Decorate it with `@risicare.agent(name=..., role=...)`.
   This opens an agent span `agent:{name}`. Use an `AgentRole` value for `role`: `orchestrator`,
   `worker`, `supervisor`, `specialist`, `router`, `aggregator`, `broadcaster`, `critic`,
   `planner`, `executor`, `retriever` or `validator`. The SDK does not check the value.
2. If the agent id must come from the code, use `with risicare.agent_context(agent_id, ...)`.
   It sets context only and creates no span.
3. Find the entry point of one conversation turn. Decorate it with `@risicare.session(...)` or
   wrap it in `with risicare.session_context(session_id, user_id=...)`.
4. Run the Verify loop. Each decorated session call and each `session_context` block adds one
   `session.end` marker span, also for the same session id: count each one.

## Code

<!-- checked against risicare 0.5.1 -->
```python
@risicare.agent(name="planner", role="planner")
def plan(task): ...

@risicare.session(session_id_arg="sid", user_id_arg="uid")
def handle_turn(sid, uid, message): ...

with risicare.session_context(conversation_id, user_id=user_id):
    with risicare.agent_context("agent-1", agent_role="worker"):
        run_worker()
```

Async code: use `async with risicare.async_session_context(...)` and
`async with risicare.async_agent_context(...)`. `@risicare.agent` works on async functions too.

## If the code shows… add…

| If code shows | Add | Why |
|---|---|---|
| A function that runs one agent | `@risicare.agent(name=..., role=...)` | One agent span per call |
| An agent id from config or a database | `with risicare.agent_context(agent_id, agent_name=..., agent_role=...)` | Context only; the first argument is the id |
| A handler with a `session_id` argument | `@risicare.session` | Reads the argument named `session_id` (and `user_id`) |
| A handler whose id argument has another name | `@risicare.session(session_id_arg="sid", user_id_arg="uid")` | Names the arguments to read |
| A known id outside a function boundary | `with risicare.session_context(session_id, user_id=...)` | The id is required |
| No id at all, but turns must group | `@risicare.session(auto_generate=True)` | Makes a UUID per call |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Bare `@risicare.session` on a function with no `session_id` argument | No session; no error (DEBUG log only) | Pass `session_id_arg=`, `session_id=` or `auto_generate=True` |
| `risicare.session_context()` with no id | `TypeError` | Pass the session id as the first argument |
| `with risicare.agent(...)` | An error: `agent` is a decorator only | Use `agent_context` for a `with` block |
| Expecting `service_name` on agent spans | `service.name` is set only on spans outside an agent | Use the agent name to filter |
| Expecting a new agent id per call from `@agent` | The id is fixed when the decorator is applied | Use `agent_context(agent_id, ...)` when the id changes per call |
| `user_id` passed to `@session(session_id=...)` as an argument | The user id is not read when `session_id=` is given | Use `session_context(session_id, user_id=...)` |

## Verification

Run the Verify loop. Expect one more span per agent call. Expect one `session.end` marker span
per decorated session call or `session_context` block (two turns of one session give two).
