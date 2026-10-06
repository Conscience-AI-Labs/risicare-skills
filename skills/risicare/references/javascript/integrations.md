---
name: integrations-javascript
description: "Read when the code uses an LLM provider SDK or an agent framework, to choose how Risicare traces it."
---
# Integrations (JavaScript)

## Prerequisites

- Setup is done. You listed the provider and framework packages from imports and `package.json`.

## Steps

1. Find each package in the tables below. "Known defects": instrument it and tell the user the
   defects. "Unverified": instrument it, prove it with a span, and tell the user that Risicare
   does not yet test it.
2. Activation is explicit. Nothing is patched by `init()` or by an import. Python patches
   automatically; do not carry that habit across.
3. Wrap each client instance with its `patchX` function from its subpath, one time. Use the
   object that it returns everywhere. The original client stays untraced.
4. You can wrap before or after `init()`. A call made before `init()` is not traced.
5. Frameworks use a callback handler or a proxy, as in the framework table.
6. Run the Verify loop. Count one LLM span per provider call, plus the framework spans below.

## Code

<!-- checked against risicare 0.8.0 (ESM run against a loopback sink; tsc strict + verbatimModuleSyntax) -->
```ts
import Anthropic from '@anthropic-ai/sdk';
import { patchAnthropic } from 'risicare/anthropic';
import { RisicareCallbackHandler } from 'risicare/langchain';

const anthropic = patchAnthropic(new Anthropic()); // call `anthropic`, never the original
const handler = new RisicareCallbackHandler(); // one instance for each call
await chain.invoke(input, { callbacks: [handler] });
```

## If the code shows… add…

| Provider (subpath) | Call | Status | Notes |
|---|---|---|---|
| OpenAI (`risicare/openai`) | `patchOpenAI(new OpenAI())` | verified | `chat.completions.create` (also streams, `withResponse()`), `embeddings.create`. Not `responses.create`: KNOWN-ISSUE (risicare-sdk #78). Not `beta.*`, images, audio |
| Anthropic (`risicare/anthropic`) | `patchAnthropic(new Anthropic())` | verified | `messages.create` and `messages.stream()` |
| Google (`risicare/google`) | `patchGoogleAI(genAI.getGenerativeModel({ model }))` | verified | Package `@google/generative-ai`, the SDK's peer dependency. `@google/genai` was not tested: tell the user. Wraps the model, not the client. `generateContent`, `generateContentStream`. A `startChat()` session gives no span |
| Vercel AI SDK (`risicare/vercel-ai`) | `const { tracedGenerateText } = patchVercelAI(); tracedGenerateText(generateText)(args)` | known defects | Pass the function in. Tokens and cost arrive (`ai` 7.x). Tool calls are not traced. TypeScript strict: TS2345; write `tracedGenerateText(generateText as never) as unknown as typeof generateText` |
| Mistral (`risicare/mistral`) | `patchMistral(new Mistral({...}))` | verified | `chat.complete`, `chat.stream` |
| Groq, Together, Cerebras (`risicare/groq`, `/together`, `/cerebras`) | `patchGroq(client)`, `patchTogether(client)`, `patchCerebras(client)` | verified | `chat.completions.create`, also streams and `withResponse()` |
| Cohere (`risicare/cohere`) | `patchCohere(new CohereClientV2({...}))` | verified | `chat`, `chatStream` |
| Ollama (`risicare/ollama`) | `patchOllama(new Ollama({...}))` | verified | `chat`, `generate`, also with `stream: true` |
| Hugging Face (`risicare/huggingface`) | `patchHuggingFace(new InferenceClient(...))` | verified | `chatCompletion`, `chatCompletionStream`, `textGeneration`. Pass `model`: without it, the span's model is the endpoint URL |
| Bedrock (`risicare/bedrock`) | `const client = patchBedrock(new BedrockRuntimeClient({...}))`, then `client.send(new ConverseCommand({...}))` | unverified | Call `send` on the returned proxy. `Converse`, `InvokeModel`; `ConverseStream` has no token counts |

| Framework (subpath) | Call | Status | Notes |
|---|---|---|---|
| LangChain (`risicare/langchain`) | `new RisicareCallbackHandler()` in `callbacks` | known defects | One LLM span `chat_model` per model call, with tokens but no model name and no cost. Inside a span, one trace. Two handler instances on one call give two LLM spans; outside a span, also two traces |
| LangGraph (`risicare/langgraph`) | `const graph = instrumentLangGraph(compiled)`, then `graph.invoke(input)` | known defects | Use the returned object: the original graph traces nothing. `graph.invoke()` sends every LangChain span two times (tokens double); `graph.stream()` sends each one time. Do not add a `RisicareCallbackHandler` too |
| LlamaIndex (`risicare/llamaindex`) | `new RisicareLlamaIndexHandler()` plus `Settings.callbackManager.on(name, (e) => handler.onEvent(e))` for each name in `RisicareLlamaIndexHandler.EVENT_NAMES` | known defects | The handler alone traces nothing. One LLM span per call, with no model name and no tokens. Outside a span, each step is its own trace: run the query in a span |
| Instructor (`risicare/instructor`) | `Instructor({ client: patchOpenAI(new OpenAI()), mode: 'TOOLS' })` | known defects | Patch the OpenAI client: its span holds the tokens and the response id. `patchInstructor` alone gives an `instructor.create` span with no tokens and no response id. Add `patchInstructor(client)` only when the user asks for the extraction span. Pass it the client, not the factory. Then each call gives two LLM spans: tell the user |

Streaming: the caller gets the provider's own stream object. The span ends when the stream ends,
with `stream.end_reason` and `stream.chunk_count`. OpenAI, Groq, Together and Cerebras streams
give tokens only with `stream_options: { include_usage: true }`. Wrap another async iterator in
`tracedStream(source, 'name')`; it adds one span to the count. The SDK does not observe the
stream handles of Stainless `parse()`, Cohere `iterMessages` and Ollama `itr`, or Bedrock
streams: KNOWN-ISSUE (risicare-sdk #72).

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `init()` only, expecting automatic tracing | No LLM spans | Wrap each client with `patchX` |
| Calls on the original client or graph after wrapping | Not traced | Replace every use with the returned object |
| `patchOpenAI(patchOpenAI(client))`, or a patch in two modules | Two LLM spans per call: KNOWN-ISSUE (risicare-sdk #67) | Patch each client one time |
| `traceContent: true` to capture prompts | Text only from `patchOpenAI` and `patchAnthropic`; a stream gives the prompt only. Other integrations send no text | Tell the user; the project setting must also allow content |
| `patchInstructor(Instructor)` (the factory). The npm README shows `patchInstructor(instructor)` and does not say what `instructor` is | Accepted, nothing traced | Patch the OpenAI client, as in the Instructor row |
| A second `RisicareCallbackHandler` on the same call | Two LLM spans; outside a span, also two traces | Pass one handler |

## Verification

Run the Verify loop. A client is traced only when its LLM span arrives: `exportedSpans`
reaches the count that includes one span per provider call. For the four framework helpers,
report the known defects with the result.
