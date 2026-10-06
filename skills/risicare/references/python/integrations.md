---
name: integrations-python
description: "Read when the code uses an LLM provider SDK or an agent framework, to choose how Risicare traces it."
---
# Integrations (Python)

## Prerequisites

- Setup is done. You listed the provider and framework packages from imports and dependencies.

## Steps

1. Find each package in the tables below. "Verified": a span arrived from the real package on
   0.5.0 or 0.5.1. "Known defects": instrument it and tell the user the defects. "Unverified":
   instrument it, prove it with a span, and tell the user that it is not verified. "Not
   traced": tell the user, and stop for that package.
2. Activation is automatic. `risicare.init()` patches the packages that are already imported,
   and a later import is patched when it returns. The import order does not matter. You write
   no patch call.
3. Bedrock is patched when `botocore.client` loads (`import boto3` does). Vertex AI is patched
   when `vertexai.generative_models` loads. Until then the state is `pending`.
4. Install an extra only when the project does not already depend on the package:
   `pip install 'risicare[langchain]'`.
5. Run the Verify loop. Count one LLM span per provider call, plus the framework spans below.

## Code

<!-- checked against risicare 0.5.1 -->
```python
import risicare
risicare.init()
import anthropic                                    # an import after init() is traced too
state = risicare.get_instrumented_modules()["anthropic"]["state"]   # "instrumented"
```

## If the code shows… add…

| Provider (module) | Status | Notes |
|---|---|---|
| OpenAI (`openai`), also OpenAI-compatible hosts by `base_url` | verified | `chat.completions.create` (also `stream=True` and the `stream()` helper) and `embeddings.create`, sync and async. Not `beta…parse`, images or audio. Not the Responses API: KNOWN-ISSUE (risicare-sdk #78). `with_raw_response` and `with_streaming_response` spans have no token counts, so no cost. With `stream=True`, `with_streaming_response` gives no span: KNOWN-ISSUE (risicare-sdk #71) |
| Anthropic (`anthropic`) | verified | `messages.create`, also `stream=True`. `messages.stream()` gives no span |
| Google Gemini (`google.generativeai`) | verified | A stream keeps `resolve()` and `text`. `google.genai` is not traced: KNOWN-ISSUE (risicare-sdk #77) |
| Vertex AI (`vertexai`) | unverified | The patch lands; no call was made against it |
| Groq (`groq`), Cerebras (`cerebras`), Together (`together`) | verified | Chat completions, also `with ... create(stream=True) as s:` |
| Cohere (`cohere`) | known defects | v1 `Client` only; `ClientV2` gives no span. With `cohere` 7.x, a v1 span holds float token counts, and the gateway refuses that whole batch (HTTP 400) |
| Ollama (`ollama`) | verified | Module functions, `Client` and `AsyncClient`. A token count that Ollama does not return is not sent |
| Hugging Face (`huggingface_hub`) | verified | `InferenceClient` `chat_completion` (also `stream=True`) and `text_generation` |
| Bedrock (`botocore`) | verified, except streams | bedrock-runtime `Converse` and `InvokeModel`. `InvokeModel` spans have no token counts: KNOWN-ISSUE (risicare-sdk #76). `ConverseStream` and `InvokeModelWithResponseStream` are unverified |
| Mistral (`mistralai`) | verified, except streams | `chat.complete`, sync and async. `chat.stream` gives no span |

| Framework (module) | Status | Notes |
|---|---|---|
| LangChain (`langchain`, `langchain_core`) | verified | One LLM span per model call. Its spans join the enclosing trace, session and agent. `stream()` and `astream()` work; with the `langchain-openai` defaults their LLM spans have no token counts. A classic `AgentExecutor` run (`langchain-classic`) is one trace, with the executor span as its root. Manual handler: `from risicare.integrations.langchain import RisicareCallbackHandler` |
| LangGraph (`langgraph`) | verified | As LangChain, plus one graph span per `invoke()`, `stream()` or `astream()` |
| LlamaIndex (`llama_index`) | known defects | Its spans join the enclosing trace, session and agent. Outside a trace, each top-level step (index build, query) is its own trace. One chat call gives two `llamaindex.llm` spans. Suppresses the provider span, except in a streamed LLM call. That call also gets a provider span, in the same trace only inside a Risicare span: KNOWN-ISSUE (risicare-sdk #82) |
| Instructor (`instructor`) | known defect | Two `instructor.create/<Model>` spans per call, plus the provider span |
| Pydantic AI (`pydantic_ai`) | known defects | `run_sync()`: two nested agent spans per run, each with the run's token totals, plus the provider spans. `run_stream()`: one agent span plus the provider spans. Tools have no span: KNOWN-ISSUE (risicare-sdk #84) |
| CrewAI (`crewai`) | known defect | `Crew.kickoff`, `kickoff_async` and agent task runs. A crew span, one span per agent, plus the provider spans. CrewAI 1.15 calls OpenAI through `with_raw_response`, so its LLM spans have no token counts and no cost |
| AutoGen (`autogen_agentchat`) | verified | `team.run()` gives one `autogen.agent/<name>/turn` span per turn. A bare `import autogen_agentchat` logs an `attached-inert` WARNING until `autogen_agentchat.agents` is imported |
| OpenAI Agents SDK (`agents`) | verified | Any module named `agents`, also the user's own, triggers it. One agent span per run, plus the provider spans |
| LiteLLM (`litellm`) | verified | Suppresses the provider span. A fallback gives one error span and one ok span, in one trace. A stream span is made after the stream ends: KNOWN-ISSUE (risicare-sdk #58) |
| DSPy (`dspy`) | verified | One `dspy.lm` span per LM call, plus module spans. A WARNING that the `litellm` hooks are `attached-inert` is expected |
| OpenTelemetry bridge (`init(otel_bridge=True)`) | known defect | With `opentelemetry-sdk` 1.45 and an SDK `TracerProvider`, every OpenTelemetry span end raises `AttributeError`. Without a `TracerProvider`, nothing is bridged. Do not turn it on. Tell the user |

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `from ollama import chat` before `init()` | That bound name stays untraced | Call `ollama.chat(...)` through the module |
| Reading `get_instrumented_modules()` as proof | `instrumented` means that a patch landed. `attempted` means that the SDK cannot tell. `pending` and `attached-inert` mean no hook is on the call path yet | Prove tracing only by a span that arrives |
| Prompt text expected in spans | The span has no prompt or completion attribute (default) | Needs `trace_content=True` and the project content setting |
| User data in a tool name | Tool-call names, the count and the finish reason are sent also with content off | Keep user data out of tool names |
| A generator that holds an open Risicare block (trace, session, agent, phase), not read to its end (`async for ... break`) | Later spans in that task join one trace: KNOWN-ISSUE (risicare-sdk #64) | Read it to its end inside the span, or close it there (`contextlib.aclosing`) |
| A provider stream not read to its end and not closed (`for ... break`) | No LLM span, and `flush()` still returns `True`: KNOWN-ISSUE (risicare-sdk #65) | Read the stream to its end, or call `stream.close()` |

## Verification

Run the Verify loop. A provider is traced only when its LLM span arrives. Count one LLM span
per provider call, plus the framework spans in the table. Wrap a LlamaIndex run in a trace to
get one trace, and report the known defects.
