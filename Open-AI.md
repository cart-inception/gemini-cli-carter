## Phased Migration Checklist (Actionable Roadmap)

Use this step-by-step checklist to migrate from Gemini to the OpenAI API, including GPT‑5 and GPT‑5 mini support. Each task lists files, expected outcomes, and dependencies. Execute phases in order.

### Phase 1 — Setup & Dependencies

- [x] Add OpenAI SDK dependency to core workspace
  - Files: packages/core (package.json via package manager)
  - Outcome: openai library available for adapters (Node SDK v4)
  - Depends on: None
- [x] (Optional) Add tokenizer for token estimation (tiktoken or gpt-tokenizer)
  - Files: packages/core (package.json)
  - Outcome: Local token estimation available for countTokens; compression can work without server counts
  - Depends on: OpenAI SDK install
- [x] Create provider folder structure for OpenAI
  - Files: packages/core/src/providers/openai/ (new directory)
  - Outcome: Place to implement OpenAIContentGenerator and helpers
  - Depends on: None
- [x] Validation: ensure repo builds unchanged after adding deps
  - Files: N/A
  - Outcome: TypeScript compile succeeds; no runtime changes yet
  - Depends on: Dep installs

### Phase 2 — Core Adapter Implementation (OpenAIContentGenerator)

- [ ] Implement OpenAIContentGenerator class that implements ContentGenerator
  - Files: packages/core/src/providers/openai/openaiContentGenerator.ts (new)
  - Outcome: Methods implemented: generateContent, generateContentStream, countTokens (estimate), embedContent, userTier?
  - Depends on: Phase 1
- [ ] Implement mapping: internal Content/PartListUnion → OpenAI messages[]
  - Files: same as above (helpers within adapter)
  - Outcome: Text parts map to assistant/user content; functionResponse parts map to role:"tool" messages with tool_call_id
  - Depends on: Adapter skeleton
- [ ] Implement mapping: Tool declarations → OpenAI tools, and tool_calls → internal functionCall parts
  - Files: same as above
  - Outcome: Model-called tools produce ToolCallRequest events (via synthesized GenerateContentResponse.functionCalls)
  - Depends on: Adapter skeleton, message mapping
- [ ] Implement streaming assembly
  - Files: same as above
  - Outcome: For stream:true, accumulate deltas and yield GenerateContentResponse-like chunks: candidates[0].content.parts[].text and functionCalls[]; map finish_reason
  - Depends on: Message mapping
- [ ] Implement JSON structured outputs support
  - Files: same as above
  - Outcome: When responseSchema/responseMimeType: 'application/json' is provided, call with response_format: { type:'json_schema', ... } and return valid JSON text content
  - Depends on: Adapter skeleton
- [ ] Implement embeddings support
  - Files: same as above
  - Outcome: embedContent uses client.embeddings.create; returns number[][] aligned to inputs
  - Depends on: Adapter skeleton
- [ ] Implement countTokens estimation
  - Files: same as above
  - Outcome: Return { totalTokens } when possible; otherwise undefined (so compression can skip gracefully)
  - Depends on: Tokenizer dep (optional)
- [ ] Validation: unit tests for adapter mappings
  - Files: packages/core/src/providers/openai/__tests__/*.test.ts (new)
  - Outcome: Tests cover messages mapping, tools, streaming assembly, JSON mode, embeddings, token counting
  - Depends on: Adapter implemented

### Phase 3 — Factory & Auth Wiring

- [ ] Add new auth type for OpenAI
  - Files: packages/core/src/core/contentGenerator.ts (AuthType enum)
  - Outcome: AuthType.USE_OPENAI = 'openai-api-key'
  - Depends on: None
- [ ] Extend createContentGeneratorConfig to support OPENAI_API_KEY
  - Files: packages/core/src/core/contentGenerator.ts
  - Outcome: When authType is USE_OPENAI, set apiKey from process.env.OPENAI_API_KEY and preserve model/proxy
  - Depends on: New AuthType
- [ ] Extend createContentGenerator to instantiate OpenAIContentGenerator
  - Files: packages/core/src/core/contentGenerator.ts
  - Outcome: Factory returns the adapter instance for USE_OPENAI
  - Depends on: Adapter class (Phase 2)
- [ ] Validate auth in CLI for OpenAI
  - Files: packages/cli/src/config/auth.ts
  - Outcome: validateAuthMethod(AuthType.USE_OPENAI) requires OPENAI_API_KEY
  - Depends on: New AuthType
- [ ] Auto-detect OpenAI in non-interactive mode
  - Files: packages/cli/src/validateNonInterActiveAuth.ts
  - Outcome: getAuthTypeFromEnv() returns USE_OPENAI when OPENAI_API_KEY is present
  - Depends on: New AuthType
- [ ] Validation: smoke test a simple prompt via CLI with OPENAI_API_KEY
  - Files: N/A
  - Outcome: Non-streaming and streaming calls return assistant content; no tool use
  - Depends on: All tasks above

### Phase 4 — GPT‑5 Model Support & Defaults

- [ ] Add OpenAI model defaults
  - Files: packages/core/src/config/models.ts
  - Outcome: DEFAULT_OPENAI_FLAGSHIP_MODEL = 'gpt-5'; DEFAULT_OPENAI_MODEL = 'gpt-5-mini'; DEFAULT_OPENAI_EMBEDDING_MODEL = 'text-embedding-3-small'
  - Depends on: None
- [ ] Use OpenAI defaults when USE_OPENAI is selected
  - Files: packages/cli/src/config/config.ts
  - Outcome: If selectedAuthType is USE_OPENAI and no explicit model is set, default to 'gpt-5-mini' (with 'gpt-5' offered as an option)
  - Depends on: Defaults added
- [ ] Surface model options (where applicable)
  - Files: Any model picker/settings use sites in CLI (e.g., packages/cli/src/config/settings.ts comments, docs)
  - Outcome: Users see “GPT‑5 (flagship)” and “GPT‑5 mini” as choices; can override with --model
  - Depends on: Defaults added
- [ ] Validation: run prompts against both 'gpt-5' and 'gpt-5-mini'
  - Files: N/A
  - Outcome: Both models run end‑to‑end (chat, streaming)
  - Depends on: Adapter + factory wiring

### Phase 5 — Integration (Streaming, Tools, JSON, Embeddings)

- [ ] Ensure Turn and UI event flow remain unchanged
  - Files: packages/core/src/core/turn.ts, packages/cli/src/ui/hooks/useGeminiStream.ts (no code edits expected)
  - Outcome: Adapter’s synthesized GenerateContentResponse drives identical ServerGeminiStreamEvent
  - Depends on: Streaming assembly complete
- [ ] Tool calling end‑to‑end
  - Files: Adapter; Turn.run() (reads functionCalls); tools infrastructure (unchanged)
  - Outcome: Model tool_calls → ToolCallRequest events → tool execution → follow‑up request contains functionResponse parts; adapter maps them to role:'tool'
  - Depends on: Tools mapping
- [ ] Structured JSON outputs path
  - Files: Adapter; packages/core/src/core/client.ts (generateJson uses ContentGenerator)
  - Outcome: generateJson() returns parsed JSON; adapter used response_format json_schema
  - Depends on: JSON mode mapping
- [ ] Embeddings
  - Files: Adapter; packages/core/src/core/client.ts (generateEmbedding)
  - Outcome: Embeddings returned as number[][] length-matched to inputs
  - Depends on: Embeddings mapping
- [ ] Compression & token counts tolerance
  - Files: packages/core/src/core/client.ts (tryCompressChat), adapter
  - Outcome: If token counts undefined, compression skipped unless forced; no crashes
  - Depends on: countTokens estimate
- [ ] Validation: targeted E2E checks
  - Files: N/A
  - Outcome:
    - Streaming text appears progressively in CLI
    - Tool calls execute and results are fed back correctly
    - JSON responses parse without code fences
    - Embeddings shape validated

### Phase 6 — Testing & Verification

- [ ] Add unit tests for adapter (messages, tools, streaming, JSON, embeddings)
  - Files: packages/core/src/providers/openai/__tests__/*.test.ts
  - Outcome: High-confidence adapter coverage
  - Depends on: Adapter
- [ ] Stabilize existing tests
  - Files: tests referencing @google/genai mocks
  - Outcome: Either keep Gemini tests intact or refactor to target ContentGenerator and adapter; do not regress non-OpenAI paths
  - Depends on: Adapter stable
- [ ] CLI non-interactive tests
  - Files: packages/cli/src/nonInteractiveCli.ts (behavioral tests)
  - Outcome: Non-interactive flow handles streaming, tool calls, and completions
  - Depends on: Integration complete
- [ ] Lint/type/build pipeline
  - Files: CI/build configs
  - Outcome: All checks green
  - Depends on: All earlier phases

### Phase 7 — Documentation & Deployment

- [ ] Update docs and README
  - Files: Open-AI.md (this file), README.md, docs/*
  - Outcome: Clear instructions for OPENAI_API_KEY, model options (GPT‑5, GPT‑5 mini), usage examples
  - Depends on: Implementation complete
- [ ] Release notes & versioning
  - Files: CHANGELOG / release docs
  - Outcome: Communicate migration and new model support
  - Depends on: Tests passing
- [ ] Operational validation
  - Files: N/A
  - Outcome: Smoke test in target environments (proxy, sandbox) with GPT‑5 models
  - Depends on: Deployment prep


## OpenAI Migration Guide for this CLI

This document is an implementation plan to switch the CLI from Google Gemini (@google/genai) to the OpenAI API and models. It is based on a scan of the current codebase. No changes have been made yet.

### Goals
- Replace Gemini model calls with OpenAI equivalents (chat, streaming, tool/function calling, JSON-structured outputs, embeddings).
- Minimize invasive changes by introducing an adapter that conforms to existing abstractions where possible.
- Preserve the CLI UX and streaming UI with minimal or no changes.

---

## 1) Current Architecture and Integration Points

Core integration is through @google/genai types and a ContentGenerator interface. Key files:

- packages/core/src/core/contentGenerator.ts
  - Defines ContentGenerator (generateContent, generateContentStream, countTokens, embedContent)
  - Factory createContentGenerator() currently constructs:
    - GoogleGenAI(models) when AuthType is USE_GEMINI or USE_VERTEX_AI
    - CodeAssistServer for Google OAuth (LOGIN_WITH_GOOGLE / CLOUD_SHELL)
  - Uses @google/genai types throughout (GenerateContentParameters, GenerateContentResponse, Part, Content, etc.)

- packages/core/src/core/geminiChat.ts
  - Chat/session orchestration built around @google/genai response shapes.
  - Handles streaming via generateContentStream; emits CLI-level stream events.
  - Appends functionResponse parts and handles automaticFunctionCallingHistory (Gemini-specific).

- packages/core/src/core/client.ts (GeminiClient)
  - Orchestrates turns, compression, JSON generation, embeddings via ContentGenerator.
  - Emits ServerGeminiStreamEvent events (provider-agnostic in shape, but named Gemini*). Uses getResponseText utilities.

- packages/core/src/utils/generateContentResponseUtilities.ts and partUtils.ts
  - Parse GenerateContentResponse (Gemini shape) into text and function calls.

- packages/core/src/core/turn.ts
  - Converts streaming responses into CLI events, expects:
    - response.candidates[0].content.parts[].text
    - response.functionCalls[] (Gemini)
    - finishReason.

- packages/cli/src/ui/hooks/useGeminiStream.ts
  - UI hook consuming ServerGeminiStreamEvent, not coupled to Gemini wire protocol, but error/fallback strings and constants reference Gemini.

- Config, models and auth:
  - packages/core/src/config/config.ts: Config; creates GeminiClient, holds ContentGeneratorConfig, model, etc.
  - packages/core/src/config/models.ts: Defaults are Gemini-specific.
  - packages/cli/src/config/auth.ts and validateNonInterActiveAuth.ts: AuthType is Gemini-centric; looks for GEMINI_API_KEY, GOOGLE_API_KEY, GOOGLE_*.

What this means:
- The abstraction boundary is ContentGenerator. If we implement an OpenAI-backed ContentGenerator that converts to/from the existing data model (Content, Part, GenerateContentResponse), the rest of the system can remain largely unchanged.
- Some Gemini-only features ("thought" parts, automaticFunctionCallingHistory, countTokens endpoint, finish reason enums) need mapping or graceful degradation.

---

## 2) OpenAI API: Concepts to Map

- SDK: openai (Node v4). Typical usage:
  - Chat Completions: client.chat.completions.create({ model, messages, tools, tool_choice, stream })
  - Streaming: set stream: true to get an async iterator of deltas.
  - Tool (function) calling: declare tools (name + JSON schema), model returns tool_calls; you must send tool results as role:"tool" messages with tool_call_id.
  - JSON structured outputs: response_format with type:"json_schema" and json_schema, or type:"json_object" (less strict). See: platform.openai.com/docs/guides/structured-outputs
  - Embeddings: client.embeddings.create({ model, input })
  - Usage: tokens reported in responses; no public token-counting endpoint. Use a tokenizer (e.g., tiktoken) for preflight estimation.

Key differences from Gemini you must handle:
- History/message format (OpenAI uses messages[]; Gemini uses Content{role, parts}).
- Tool result return messages (OpenAI role:"tool" with tool_call_id vs Gemini functionResponse parts in a user message).
- Streaming payload shape (OpenAI choices[].delta vs Gemini candidates[].content.parts).
- No SDK-native countTokens; you’ll estimate tokens locally or use usage from a reply.
- Finish reasons differ (map to your internal enum).

---

## 3) Strategy: Adapter-Based Provider

Recommended approach: Add an OpenAIContentGenerator that implements the existing ContentGenerator interface and performs all necessary translations.

Pros:
- Minimizes changes across CLI, chat loop, tools, and UI.
- Allows gradual provider switching behind config/AuthType.

Scope to cover in the adapter:
- Input translation: Content/PartListUnion → OpenAI messages + attachments.
- Tools: FunctionDeclaration[] → OpenAI tools; generate tool call requests; convert tool results from current functionResponse parts to role:"tool" messages.
- Streaming: Map OpenAI streamed deltas to synthesized GenerateContentResponse chunks that look like Gemini responses for downstream consumers.
- JSON mode: Map SchemaUnion to JSON Schema; set response_format; parse model output to JSON.
- Finish reasons: map OpenAI finish_reason to @google/genai FinishReason (best-effort).
- Usage: populate usageMetadata (total tokens etc.) from OpenAI response usage.
- Embeddings: map to numbers[][].
- countTokens: implement using tiktoken/gpt-tokenizer estimate; return undefined on failure to avoid breaking compression.

---

## 4) Implementation Plan (by file)

Add provider + config/auth wiring; keep Gemini support until fully switched.

1) Add OpenAI dependency to core
- In packages/core: `npm -w packages/core install openai`
- Optionally: `npm -w packages/core install gpt-tokenizer` (or @dqbd/tiktoken) for token estimation.

2) Extend Auth types and CLI detection
- packages/core/src/core/contentGenerator.ts
  - Add enum member: USE_OPENAI = 'openai-api-key'
  - ContentGeneratorConfig: add provider switch (or infer from auth type).
  - createContentGeneratorConfig: if process.env.OPENAI_API_KEY set and selected auth is USE_OPENAI, set apiKey and provider flag.
- packages/cli/src/config/auth.ts
  - validateAuthMethod: when AuthType.USE_OPENAI, require OPENAI_API_KEY.
- packages/cli/src/validateNonInterActiveAuth.ts
  - getAuthTypeFromEnv(): detect OPENAI_API_KEY → AuthType.USE_OPENAI

3) Create OpenAI adapter
- New: packages/core/src/providers/openai/openaiContentGenerator.ts
  - import OpenAI from 'openai'
  - class OpenAIContentGenerator implements ContentGenerator
    - constructor(apiKey, model, httpOptions? proxy)
    - generateContent(req, userPromptId)
    - generateContentStream(req, userPromptId)
    - countTokens(req) → estimate
    - embedContent(req)
    - userTier?: leave undefined or set pseudo-tier
  - Helpers:
    - mapHistoryToOpenAIMessages(history: Content[], newUserMessage: PartListUnion)
      - Convert Content.parts:
        - text → { type: 'text', text }
        - functionResponse → translate to role:'tool' message with tool_call_id (requires preserving fn id)
        - functionCall parts in model responses map back from OpenAI tool_calls to synthesized parts with functionCall
    - mapToolsToOpenAI(functionDeclarations)
    - synthesizeGenerateContentResponseFromOpenAI(message|delta, extras)
      - Produce { candidates:[{ content:{ role:'model', parts:[{text: ...} or {functionCall:...}]}, finishReason, index:0, safetyRatings:[] }], usageMetadata }
      - For streaming, accumulate content delta text and tool_call deltas to emit chunk objects consistent with downstream expectations.

4) Factory changes
- packages/core/src/core/contentGenerator.ts
  - In createContentGenerator(), add branch for AuthType.USE_OPENAI to instantiate OpenAIContentGenerator.
  - Keep Google branches intact for now.

5) JSON structured outputs
- packages/core/src/core/client.ts generateJson()
  - The adapter must honor responseSchema / responseMimeType by:
    - Providing system+user messages and setting response_format to json_schema with the provided SchemaUnion converted to JSON Schema.
    - Returning parsed JSON string → parsed object as today.

6) Tool calling behavior in Turn and UI
- Current flow:
  - Turn.run() pulls resp.functionCalls and yields ToolCallRequest events.
  - UI/tooling executes tools and replies with a "user" message that contains functionResponse parts.
- Adapter requirement:
  - When it sees functionResponse parts in the next turn request, emit OpenAI tool result messages (role:'tool') with the matching tool_call_id.
  - Maintain a mapping between function call IDs and OpenAI tool_call_ids across the turn.

7) Embeddings
- GeminiClient.generateEmbedding() calls contentGenerator.embedContent.
- Adapter: use client.embeddings.create; return arrays of numbers; ensure the response length matches input length.

8) Token counting & compression
- tryCompressChat() calls contentGenerator.countTokens for curated history.
- Adapter: implement countTokens by estimating tokens for the target model using gpt-tokenizer/tiktoken. If estimation is unavailable, return { totalTokens: undefined } so compression is skipped unless forced.

9) Defaults and models
- packages/core/src/config/models.ts
  - Add OpenAI defaults:
    - DEFAULT_OPENAI_MODEL: e.g., 'gpt-4o-mini' (chat)
    - DEFAULT_OPENAI_EMBEDDING_MODEL: e.g., 'text-embedding-3-small'
  - Config.getModel() will continue to return current model string; initial value in CLI config should be switched to DEFAULT_OPENAI_MODEL when using OpenAI.

10) Error handling and finish reasons
- Map OpenAI finish_reason to @google/genai FinishReason:
  - 'stop' → FinishReason.STOP
  - 'length' → FinishReason.MAX_TOKENS
  - 'content_filter' → FinishReason.SAFETY (best-effort)
  - tool-related → keep emitting ToolCallRequest instead of Finished or emit Finished with reason OTHER after tool batch
- Rate limit behavior: replace 429 fallback logic with OpenAI-specific handling (no "Flash" fallback). Consider model fallback list (e.g., gpt-4o → gpt-4o-mini) behind a handler.

11) Telemetry & logging
- The telemetry logs content and timing based on GenerateContentResponse; no provider changes needed if the adapter fills usageMetadata.

12) Tests
- Add unit tests for the adapter mapping (messages, tools, streaming, JSON mode, embeddings, token estimation).
- Update existing tests that vi.mock('@google/genai') to instead mock the adapter or keep as-is if Gemini remains for tests.
- Ensure CLI non-interactive path (packages/cli/src/nonInteractiveCli.ts) still works: it consumes ServerGeminiStreamEvent only.

---

## 5) Data Model Mapping Cheatsheet

Internal (Gemini) → OpenAI Chat Completions

Messages/History:
- Content { role:'user', parts:[{text}...] } → message { role:'user', content: string | array of segments }
- Content { role:'model', parts:[{text}...] } → message { role:'assistant', content: string }
- Function call (from model): part.functionCall { name, args, id } → assistant tool_calls: [{ id, type:'function', function:{ name, arguments: JSON.stringify(args) } }]
- Function response (from tools): part.functionResponse { name, response: {name, content} } → message { role:'tool', tool_call_id: <id>, content: stringified content }

Tools (function declarations):
- FunctionDeclaration { name, description, parametersJsonSchema | parameters } → tools: [{ type:'function', function: { name, description, parameters: JSONSchema } }]

Streaming:
- OpenAI stream yields deltas per choice; accumulate delta.content and delta.tool_calls[].function.arguments; on each event, synthesize a GenerateContentResponse chunk with updated parts[].text or parts[].functionCall text (arguments string) as appropriate.

Finish reasons:
- Map OpenAI finish_reason → FinishReason enum as above.

Usage:
- response.usage.prompt_tokens|completion_tokens|total_tokens → usageMetadata

JSON mode:
- response_format: { type:'json_schema', json_schema: { name:'Result', schema: <SchemaUnion-converted> } }
- Strict: also set: temperature=0; and parse content to JSON (assistant message content).

Embeddings:
- client.embeddings.create({ model, input: string[] }) → data[].embedding (number[]) → return as number[][]

---

## 6) Step-by-Step Execution Outline

1) Add new AuthType and settings plumbing:
- Add AuthType.USE_OPENAI and OPENAI_API_KEY validation; auto-detect in non-interactive mode when OPENAI_API_KEY is set.

2) Implement OpenAIContentGenerator:
- Constructor wires OpenAI SDK with apiKey and optional proxy via environment (the SDK respects HTTPS_PROXY) or pass fetch/dispatcher.
- Implement generateContent():
  - Convert history+new message to messages[].
  - Add tools if present.
  - Call client.chat.completions.create({ model, messages, tools, tool_choice:'auto' }).
  - Convert to GenerateContentResponse-like object for consumers.
- Implement generateContentStream():
  - Same as above but with stream:true, iterate deltas, yield synthesized chunks.
- Implement embedContent():
  - client.embeddings.create({ model: embeddingModel, input: texts }).
- Implement countTokens():
  - Estimate with tokenizer by concatenating mapped messages for the selected model into a single prompt context.

3) Factory and config:
- createContentGenerator(): add branch for USE_OPENAI returning new OpenAIContentGenerator.
- createContentGeneratorConfig(): when USE_OPENAI, set apiKey from OPENAI_API_KEY and set model defaults (DEFAULT_OPENAI_MODEL if none).
- CLI config default model: point to DEFAULT_OPENAI_MODEL when selectedAuthType is USE_OPENAI.

4) Compatibility behaviors:
- Thought chunks: if using standard OpenAI models, omit "thought" events. Optionally, support OpenAI Reasoning models and map their reasoning content to Thought events.
- automaticFunctionCallingHistory: leave empty; continue to update history based on assistant/user/tool messages mapped at the adapter boundary.

5) Test & verify locally:
- Unit tests for mapping functions (history ↔ messages, tools, streaming assembly, JSON mode, embeddings).
- Run existing CLI integration tests; verify streaming, tool calls (end-to-end), JSON generation, embeddings, compression not crashing when token estimate undefined.

---

## 7) Configuration & Environment Variables

- OPENAI_API_KEY: required for AuthType.USE_OPENAI
- Optional model overrides via CLI flag or settings.json; recommended defaults:
  - DEFAULT_OPENAI_MODEL: gpt-4o-mini (fast, tool-use capable, vision).
  - DEFAULT_OPENAI_EMBEDDING_MODEL: text-embedding-3-small
- Proxy: existing proxy config (HTTPS_PROXY etc.) should continue to work; the OpenAI SDK uses fetch under the hood.

---

## 8) Risks, Gaps, and Mitigations

- Tool response shape mismatch: Our current pipeline sends functionResponse parts inside a user message; OpenAI expects role:'tool'. Mitigate within adapter by translating outgoing functionResponse parts into a sequence of appropriate tool messages.
- Token counting: No native endpoint; use estimators; compression will be skipped if token counts are undefined and not forced.
- Thought streaming: Gemini-specific; either omit or map from OpenAI Reasoning API when/if adopted.
- Finish reason coverage: Some Gemini finish reasons (e.g., LANGUAGE, PROHIBITED_CONTENT) have no direct OpenAI analog; map to OTHER or SAFETY as best-effort.
- Tests mock @google/genai in places: keep Gemini support during transition or refactor tests to target the ContentGenerator abstraction and the new adapter.

---

## 9) Concrete Touchpoints Checklist

Update or add the following (paths are relative to repo root):
- packages/core/src/core/contentGenerator.ts (AuthType, factory branches, config)
- packages/core/src/providers/openai/openaiContentGenerator.ts (new)
- packages/cli/src/config/auth.ts (validate OPENAI_API_KEY)
- packages/cli/src/validateNonInterActiveAuth.ts (env detection)
- packages/core/src/config/models.ts (DEFAULT_OPENAI_* constants)
- packages/cli/src/config/config.ts (default model selection when using OpenAI)
- Optionally: docs and README to reflect OpenAI usage

No changes expected in:
- packages/cli/src/ui/hooks/useGeminiStream.ts
- packages/core/src/core/turn.ts and client.ts
…provided the adapter faithfully synthesizes GenerateContentResponse chunks.

---

## 10) Example: Streaming Adapter Pseudocode

```ts
for await (const delta of oaiStream) {
  // accumulate assistant text
  if (delta.choices?.[0]?.delta?.content) {
    buffer += delta.choices[0].delta.content as string;
    yield synthesizeChunk({ textDelta: delta.choices[0].delta.content });
  }
  // accumulate tool call deltas
  if (delta.choices?.[0]?.delta?.tool_calls) {
    // build up pending tool_calls by id; when complete, yield a chunk with functionCall part
  }
  // when finish_reason present, yield a final chunk with mapped finishReason
}
```

---

## 11) Example: Tools Mapping

```ts
// From internal FunctionDeclaration[] to OpenAI tools
const tools = declarations.map(d => ({
  type: 'function',
  function: {
    name: d.name,
    description: d.description,
    parameters: d.parametersJsonSchema ?? d.parameters, // ensure valid JSON Schema
  },
}));

// From OpenAI tool_calls to internal functionCall parts
// tool_call = { id, function: { name, arguments } }
const part = { functionCall: { id, name, args: JSON.parse(arguments) } };
```

---

## 12) Example: JSON Structured Output

```ts
const response = await client.chat.completions.create({
  model,
  messages,
  response_format: {
    type: 'json_schema',
    json_schema: {
      name: 'Result',
      schema: yourJsonSchema, // from SchemaUnion
      strict: true,
    },
  },
  temperature: 0,
});
const text = response.choices[0].message.content;
const obj = JSON.parse(text!);
```

---

## 13) Rollout Plan

- Phase 1: Introduce OpenAI adapter side-by-side with existing Gemini path; behind AuthType.USE_OPENAI. Add tests.
- Phase 2: Switch CLI defaults to OpenAI when OPENAI_API_KEY present; keep Gemini as optional.
- Phase 3: Remove Gemini-specific branches when desired, update naming (GeminiClient → LLMClient) in a separate refactor.

---

## GPT‑5 and GPT‑5 mini (new, just released)

This CLI can target OpenAI’s newest GPT‑5 family. Based on OpenAI’s announcements today (see References), GPT‑5 is the flagship successor to the GPT‑4o line, and “GPT‑5 mini” is a smaller, faster, lower‑cost variant. Integrating these into this app follows the same adapter plan above with only model IDs changing.

Key points and recommendations:
- Model IDs
  - Use 'gpt-5' for the flagship model
  - Use 'gpt-5-mini' for the small/fast variant
  - Note: Always verify the exact model identifiers in the OpenAI API reference for your account/region before deployment.
- Capabilities
  - Chat Completions API (streaming and non‑streaming)
  - Tool/function calling with JSON schema parameters and automatic tool_calls
  - Structured JSON outputs via response_format: { type: 'json_schema', ... }
  - Vision/multimodal support depending on the specific SKU; confirm in API docs
  - Usage/limits exposed via response.usage
- Thinking/Reasoning
  - OpenAI’s launch materials indicate enhanced “thinking” capabilities. If the SDK surfaces reasoning content or metadata, optionally map those to the existing “Thought” event in the UI. Otherwise, fall back to standard assistant content.
- Token counting
  - No server‑side token count API; continue local estimation (e.g., tiktoken). If unavailable for the new models initially, return undefined to skip compression heuristics.
- Embeddings
  - Keep embeddings on embedding models (e.g., text-embedding-3-small). GPT‑5 chat models are typically not used for embeddings.

Adapter usage (unchanged except model name):
- Non‑streaming
  - client.chat.completions.create({ model: 'gpt-5', messages, tools })
- Streaming
  - client.chat.completions.create({ model: 'gpt-5-mini', messages, tools, stream: true }) and assemble deltas into GenerateContentResponse‑like chunks.

CLI configuration changes:
- packages/core/src/config/models.ts
  - Add new defaults specifically for OpenAI:
    - DEFAULT_OPENAI_FLAGSHIP_MODEL = 'gpt-5'
    - DEFAULT_OPENAI_MODEL = 'gpt-5-mini' (recommended default for most sessions)
    - DEFAULT_OPENAI_EMBEDDING_MODEL = 'text-embedding-3-small'
- packages/cli/src/config/config.ts
  - When AuthType.USE_OPENAI is selected, default to DEFAULT_OPENAI_MODEL (gpt‑5‑mini) unless overridden by user settings or --model flag.
- Model picker / settings (where surfaced to users)
  - Offer two primary OpenAI options: “GPT‑5 (flagship)” and “GPT‑5 mini (fast/cost‑effective)”.

Tool/function calling with GPT‑5 (unchanged mapping):
- Declare tools from our FunctionDeclaration[] → OpenAI tools [{ type:'function', function: { name, parameters } }]
- When the model returns tool_calls (with id + function arguments), emit ToolCallRequest events as today
- When tools complete, convert functionResponse parts in the follow‑up request to role:'tool' messages paired with the correct tool_call_id

Structured JSON outputs with GPT‑5 (unchanged mapping):
- Use response_format: { type:'json_schema', json_schema:{ name:'Result', schema, strict:true } } and temperature: 0; parse assistant content as JSON

Error handling and finish reasons:
- Map OpenAI finish_reason to our internal FinishReason as described earlier; fall back to OTHER when no direct mapping exists

Validation checklist for GPT‑5 enablement:
- [ ] OPENAI_API_KEY configured
- [ ] AuthType.USE_OPENAI selected or auto‑detected
- [ ] DEFAULT_OPENAI_* constants added and wired
- [ ] Adapter tested against both 'gpt-5' and 'gpt-5-mini' for:
  - [ ] streaming deltas → UI
  - [ ] tool_calls end‑to‑end
  - [ ] JSON structured outputs
  - [ ] generateJson() path
  - [ ] compression path tolerant to undefined token counts


## 14) References
- OpenAI API: https://platform.openai.com/docs/api-reference
- Structured outputs: https://platform.openai.com/docs/guides/structured-outputs
- Node SDK: https://github.com/openai/openai-node

End of plan.

