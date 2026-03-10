# TypeScript Integration Playbook

Complete guide for integrating ZeroEval into a TypeScript/JavaScript AI application.

## Install and Initialize

### 1. Install the SDK

```bash
npm install zeroeval
```

Requires Node 18+ (also works with Bun and browser builds via Vite/Next.js).

Install peer dependencies for integrations you need:

```bash
npm install openai                # OpenAI integration
npm install ai @ai-sdk/openai    # Vercel AI SDK integration
npm install langchain             # LangChain integration
```

### 2. Get Your API Key

If you don't have an API key yet, generate one from the ZeroEval dashboard:

1. Go to [app.zeroeval.com/settings?section=project-api-keys](https://app.zeroeval.com/settings?section=project-api-keys).
2. Click **New Key**.
3. Enter a name (e.g. "dev-laptop") and pick an expiration (or "Never expires" for local dev).
4. Click **Create Key**.
5. Copy the key immediately -- it's only shown once.

Keys are scoped to a single project. If you work across multiple projects, generate a key for each one.

### 3. Configure the API Key

**Option A — Environment variable (recommended):**

```bash
export ZEROEVAL_API_KEY=sk_ze_your_key_here
```

The SDK auto-initializes on the first span when `ZEROEVAL_API_KEY` is set.

**Option B — In code:**

```typescript
import * as ze from "zeroeval";
ze.init({ apiKey: "sk_ze_your_key_here" });
```

### 4. Initialize the SDK

```typescript
import * as ze from "zeroeval";

ze.init();
```

`ze.init()` accepts these optional parameters:

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `apiKey` | `string` | `ZEROEVAL_API_KEY` env var | API authentication |
| `apiUrl` | `string` | `https://api.zeroeval.com` | API endpoint (override for local dev only) |
| `workspaceName` | `string` | `"Personal Workspace"` | Workspace display name |
| `debug` | `boolean` | `false` | Verbose logging (`ZEROEVAL_DEBUG=true` also works) |
| `flushInterval` | `number` | `10000` (10s) | Span buffer flush interval in ms |
| `maxSpans` | `number` | `100` | Max buffered spans before auto-flush |
| `integrations` | `Record<string, boolean>` | All enabled | e.g. `{ openai: false }` |

---

## Verify First Trace

### OpenAI Integration

```typescript
import * as ze from "zeroeval";
import { OpenAI } from "openai";

ze.init({ debug: true });

const openai = ze.wrap(new OpenAI());

const completion = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "What is the capital of France?" }],
});

console.log(completion.choices[0].message.content);

ze.tracer.shutdown();
```

With `debug: true`, confirm that span creation and flush messages appear in the console. Check the ZeroEval dashboard for the trace.

### Vercel AI SDK Integration

```typescript
import * as ze from "zeroeval";
import { openai } from "@ai-sdk/openai";
import * as ai from "ai";

const wrappedAI = ze.wrap(ai);

const result = await wrappedAI.generateText({
  model: openai("gpt-4o-mini"),
  prompt: "Hello, world!",
});

console.log(result.text);
ze.tracer.shutdown();
```

### LangChain / LangGraph Integration

```typescript
import {
  ZeroEvalCallbackHandler,
  setGlobalCallbackHandler,
} from "zeroeval/langchain";

setGlobalCallbackHandler(new ZeroEvalCallbackHandler());
```

Pass the callback handler explicitly to chain/graph invocations for reliable tracing. The `setGlobalCallbackHandler` convenience works for simple scripts but may not propagate in all async contexts.

### Manual Spans

For providers without auto-instrumentation, use manual spans:

```typescript
await ze.withSpan({ name: "my_pipeline" }, async () => {
  const traceId = ze.getCurrentTrace();
  if (traceId) {
    ze.setTag(traceId, { feature: "qa", user: "demo" });
  }

  const response = await myCustomLLMCall();
  return response;
});
```

### Supported Auto-Integrations

| Integration | Peer Dependency | Traced Operations |
|-------------|----------------|-------------------|
| OpenAI | `openai` | `chat.completions.create` (streaming + non-streaming), embeddings |
| Vercel AI SDK | `ai`, `@ai-sdk/openai` | `generateText`, `streamText`, `generateObject`, `embed` |
| LangChain | `langchain` | Via callback handler |

`ze.wrap()` only supports OpenAI and Vercel AI SDK clients. For all other providers, use manual spans.

---

## ze.prompt Migration

### Why Migrate

Wrapping system prompts with `ze.prompt` enables:
- Automatic version tracking and content hashing.
- A/B testing between prompt versions.
- Automated prompt optimization from feedback data.
- Prompt-linked judge evaluations.

### Placement and Resilience

`ze.prompt()` performs a network call to register or fetch prompt versions. **Never call it at module scope or with top-level `await`.** Top-level async module evaluation is fragile across runtimes and bundlers. Resolve prompts inside route handlers, server actions, agent execution functions, or jobs.

```typescript
// BAD — top-level await blocks module loading, can timeout and crash startup
const SYSTEM_PROMPT = await ze.prompt({ name: "customer-support", content: "..." });

// GOOD — resolves when the handler runs, not at import time
const DEFAULT_PROMPT = "You are a helpful customer support assistant.";

async function handleRequest(userMessage: string) {
  const systemPrompt = await ze.prompt({
    name: "customer-support",
    content: DEFAULT_PROMPT,
    from: "explicit",
  });
  // ...
}
```

For apps that must start even when the ZeroEval API is slow or unreachable, wrap the call with a fallback:

```typescript
const DEFAULT_PROMPT = "You are a helpful customer support assistant.";

async function getSystemPrompt(): Promise<string> {
  try {
    return await ze.prompt({
      name: "customer-support",
      content: DEFAULT_PROMPT,
      from: "explicit",
    });
  } catch {
    return DEFAULT_PROMPT;
  }
}

async function handleRequest(userMessage: string) {
  const systemPrompt = await getSystemPrompt();
  // ...
}
```

### Staged Rollout

**Stage 1 — Explicit mode (safe rollout — always returns your local content):**

```typescript
async function handleRequest(userMessage: string) {
  const systemPrompt = await ze.prompt({
    name: "customer-support",
    content: "You are a helpful customer support assistant.",
    from: "explicit",
  });
  // ...
}
```

This registers the prompt version without swapping content at runtime. Note: `explicit` still makes a network call to register the version — it is safe from content swapping, not from network latency.

**Stage 2 — Auto mode (default, recommended for production):**

```typescript
async function handleRequest(userMessage: string) {
  const systemPrompt = await ze.prompt({
    name: "customer-support",
    content: "You are a helpful customer support assistant.",
  });
  // ...
}
```

The SDK tries to fetch the latest optimized version. If none exists, it uses the provided content as fallback.

**Stage 3 — Latest mode (strict, requires published version):**

```typescript
async function handleRequest(userMessage: string) {
  const systemPrompt = await ze.prompt({
    name: "customer-support",
    from: "latest",
  });
  // ...
}
```

Fails if no prompt version exists in the backend.

### Template Variables

```typescript
async function handleRequest(userMessage: string, role: string, domain: string) {
  const systemPrompt = await ze.prompt({
    name: "support-agent",
    content: "You are a {{role}} assistant specializing in {{domain}}.",
    variables: { role, domain },
  });
  // ...
}
```

Variables are interpolated by the OpenAI/Vercel wrappers before the prompt reaches the LLM.

### Prompt Metadata

`ze.prompt` attaches the metadata ZeroEval needs to version, trace, and optimize prompts. Supported integrations (OpenAI, Vercel AI) handle this automatically. For unsupported providers, use manual spans to link prompts to traces.

---

## Sending Feedback

```typescript
await ze.sendFeedback({
  promptSlug: "customer-support",
  completionId: "span-uuid-here",
  thumbsUp: true,
  reason: "Response was accurate and helpful",
  expectedOutput: "Optional ideal response text",
});
```

For scored judge feedback:

```typescript
await ze.sendFeedback({
  promptSlug: "quality-scorer",
  completionId: "span-uuid-here",
  thumbsUp: false,
  judgeId: "automation-uuid-here",
  expectedScore: 3.5,
  scoreDirection: "too_high",
});
```

### Getting the Completion ID

Use the current span ID as the completion ID:

```typescript
const spanId = ze.getCurrentSpan()?.spanId;
if (spanId) {
  await ze.sendFeedback({
    promptSlug: "my-prompt",
    completionId: spanId,
    thumbsUp: true,
  });
}
```

---

## Common Pitfalls

### ze.prompt at module scope
`ze.prompt()` performs network I/O. Using top-level `await` to call it at module scope blocks module loading and can timeout or crash startup. Always resolve prompts inside route handlers, server actions, or execution functions. See "Placement and Resilience" above.

### Wrap, don't import raw
Always use `ze.wrap(new OpenAI())` instead of `new OpenAI()` directly. Without wrapping, no auto-tracing occurs.

### Flush before exit
The span writer is async and buffered. For scripts, serverless functions, or short-lived processes, always flush:

```typescript
await ze.tracer.flush();
ze.tracer.shutdown();
```

Call `flush()` before `sendFeedback()` in short-lived flows to ensure spans are ingested before feedback references them.

### API URL confusion
The SDK defaults to `https://api.zeroeval.com`. Some example files use `http://localhost:8000` for local development. Do not copy localhost URLs into production code.

### Debug mode
Pass `debug: true` to `ze.init()` or set `ZEROEVAL_DEBUG=true` to see detailed span lifecycle logs.

