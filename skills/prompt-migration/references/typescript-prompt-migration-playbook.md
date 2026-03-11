# TypeScript Prompt Migration Playbook

Step-by-step guide for migrating hardcoded TypeScript prompts to `ze.prompt`.

---

## Migration Pattern

### Before migration

```typescript
const SYSTEM_PROMPT = "You are a helpful customer support assistant.";

async function handleRequest(userMessage: string) {
  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "system", content: SYSTEM_PROMPT },
      { role: "user", content: userMessage },
    ],
  });
  return response.choices[0].message.content;
}
```

### After migration (Stage 1 -- explicit mode)

```typescript
import { OpenAI } from "openai";
import * as ze from "zeroeval";

ze.init();
const openai = ze.wrap(new OpenAI());

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
  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: userMessage },
    ],
  });
  return response.choices[0].message.content;
}
```

Key changes:

1. The hardcoded string is preserved as `DEFAULT_PROMPT` (the fallback).
2. The OpenAI client is wrapped with `ze.wrap()` for auto-tracing and metadata extraction.
3. `ze.prompt()` is called inside an async helper function, not at module scope or with top-level `await`.
4. `from: "explicit"` ensures the local content is always used, while registering the version in ZeroEval.
5. A try/catch falls back to the local string if ZeroEval is unreachable.

---

## Placement and Resilience

`ze.prompt()` is async and performs a network call. Never call it with top-level `await` or at module scope. Top-level async module evaluation is fragile across runtimes and bundlers.

```typescript
// BAD -- top-level await blocks module loading, can timeout and crash startup
const SYSTEM_PROMPT = await ze.prompt({
  name: "my-agent",
  content: "...",
});

// GOOD -- resolves when the handler runs, not at import time
const DEFAULT_PROMPT = "You are a helpful assistant.";

async function handleRequest(userMessage: string) {
  const systemPrompt = await ze.prompt({
    name: "my-agent",
    content: DEFAULT_PROMPT,
    from: "explicit",
  });
  // ...
}
```

---

## Wrapper Requirement

Automatic metadata extraction and variable interpolation only work with wrapped clients:

```typescript
// Required for OpenAI
const openai = ze.wrap(new OpenAI());

// Required for Vercel AI SDK
import * as ai from "ai";
ze.wrapVercelAI(ai);
```

Without wrapping, no auto-tracing occurs and prompt metadata is not extracted from the system message.

`ze.wrap()` supports OpenAI and Vercel AI SDK clients. For all other providers, use manual spans (see the unsupported providers section below).

---

## First System Message Rule

The ZeroEval wrappers only scan the **first** message in the messages array for metadata. That message must have `role: "system"` and contain the decorated prompt returned by `ze.prompt()`.

If the decorated prompt is not the first system message, metadata extraction will not happen automatically.

```typescript
// GOOD -- decorated prompt is the first system message
const messages = [
  { role: "system", content: systemPrompt },
  { role: "user", content: userMessage },
];

// BAD -- user message comes first, metadata will not be extracted
const messages = [
  { role: "user", content: userMessage },
  { role: "system", content: systemPrompt },
];
```

---

## Staged Rollout

### Stage 1 -- Explicit mode (safe rollout)

```typescript
const systemPrompt = await ze.prompt({
  name: "customer-support",
  content: DEFAULT_PROMPT,
  from: "explicit",
});
```

Always returns your local content. Registers the version in ZeroEval for tracking. `from: "explicit"` is safe from content swapping but still makes a network call.

### Stage 2 -- Auto mode (recommended production)

```typescript
const systemPrompt = await ze.prompt({
  name: "customer-support",
  content: DEFAULT_PROMPT,
});
```

The SDK tries to fetch the latest optimized version. If none exists, it uses the provided content as fallback.

### Stage 3 -- Latest mode (strict)

```typescript
const systemPrompt = await ze.prompt({
  name: "customer-support",
  from: "latest",
});
```

Fails if no prompt version exists in the backend.

### Hash pinning (deterministic rollback)

```typescript
const systemPrompt = await ze.prompt({
  name: "customer-support",
  from: "a1b2c3d4e5f6...",  // full 64-char SHA-256 hash
});
```

Fetches an exact version by content hash.

---

## Template Variables

Use `{{variable}}` syntax for dynamic prompt content:

```typescript
const systemPrompt = await ze.prompt({
  name: "support-agent",
  content: "You are a {{role}} assistant specializing in {{domain}}.",
  variables: { role: "billing", domain: "subscription management" },
});
```

Variables are interpolated by the OpenAI/Vercel wrappers before the prompt reaches the LLM.

---

## Wiring Feedback

Feedback is required for prompt optimization. Wire it as part of migration.

### Getting the completion ID

Use the current span ID as the completion ID:

```typescript
const spanId = ze.getCurrentSpan()?.spanId;
```

This is the safest path for TypeScript. Some docs mention `response.id` from the OpenAI response object, but the span ID is the canonical value that ZeroEval uses internally.

### Basic prompt feedback

```typescript
const spanId = ze.getCurrentSpan()?.spanId;
if (spanId) {
  await ze.sendFeedback({
    promptSlug: "customer-support",
    completionId: spanId,
    thumbsUp: false,
    reason: "Response was too verbose",
    expectedOutput: "A concise 2-3 sentence response",
  });
}
```

### Scored judge feedback

For judges that use numeric rubrics:

```typescript
await ze.sendFeedback({
  promptSlug: "quality-scorer",
  completionId: spanId,
  thumbsUp: false,
  judgeId: "automation-uuid-here",
  expectedScore: 3.5,
  scoreDirection: "too_high",
});
```

Note: the TypeScript SDK does not currently support criterion-level judge feedback (`criteriaFeedback`). For per-criterion corrections, use the Python SDK or the REST API directly.

### Flush before feedback in short-lived flows

The span writer is async and buffered. In serverless functions, scripts, or short-lived processes, flush before sending feedback:

```typescript
await ze.tracer.flush();

// Now the span exists server-side
await ze.sendFeedback({
  promptSlug: "my-prompt",
  completionId: spanId,
  thumbsUp: true,
});

// Shutdown before exit
ze.tracer.shutdown();
```

---

## Unsupported Providers

For providers that are not auto-instrumented (anything other than OpenAI and Vercel AI SDK), use manual spans and metadata extraction:

```typescript
import * as ze from "zeroeval";

const DEFAULT_PROMPT = "You are a helpful assistant.";

async function handleRequest(userMessage: string) {
  const decoratedPrompt = await ze.prompt({
    name: "my-agent",
    content: DEFAULT_PROMPT,
    from: "explicit",
  });

  const { metadata, cleanContent } = ze.extractZeroEvalMetadata(decoratedPrompt);

  const result = await ze.withSpan(
    {
      name: "llm-call",
      attributes: {
        kind: "llm",
        task: metadata?.task,
        zeroeval: metadata,
      },
    },
    async () => {
      // Pass cleanContent (without metadata tags) to the unsupported provider
      return await myProvider.complete({
        systemPrompt: cleanContent,
        userMessage,
      });
    }
  );

  return result;
}
```

---

## Connecting Judges

Linking a judge to your migrated prompt closes the optimization loop.

### Why link

- Prompt-linked judges auto-evaluate spans from that prompt.
- Judge evaluations become optimization signal alongside human feedback.
- Optimization runs can weight multiple linked judges as separate intents.

### How to link

**Dashboard**: open your project, navigate to Judges, select the judge, and link it to the prompt.

**API**:

```
POST /signals/projects/{project_id}/prompts/{prompt_id}/judges/{automation_id}/link
```

If no judges exist yet, use the `create-judge` skill to design and create one.

---

## Prompt Optimization

Once feedback is flowing and judges are linked, prompt optimization becomes available.

### How it works

1. `ze.prompt()` registers prompt versions and links them to traced spans.
2. Feedback (human or judge) accumulates against those spans.
3. When enough feedback exists, trigger optimization from the dashboard ("Optimize Prompt") or via CLI.
4. The optimizer generates candidate prompts using the feedback data.
5. Review candidates and adopt the best one. Adoption creates a new prompt version, pins the `production` tag, and updates linked judge templates.
6. After adoption, `ze.prompt()` in auto mode automatically serves the new version.

---

## Validation Checklist

After migration, verify:

- [ ] `ze.init()` completes without errors.
- [ ] `ze.wrap()` is applied to the OpenAI or Vercel AI client.
- [ ] `ze.prompt()` returns a decorated string (contains `<zeroeval>` metadata prefix).
- [ ] The traced span in the dashboard shows the prompt name and version.
- [ ] The decorated prompt is the first system message in the messages array.
- [ ] `ze.sendFeedback()` returns a feedback record without errors.
- [ ] If judges are linked, evaluations appear in the judges dashboard section.

---

## Common Pitfalls

### ze.prompt with top-level await

`ze.prompt()` performs network I/O. Using top-level `await` blocks module loading and can timeout or crash startup. Always resolve prompts inside route handlers, server actions, or execution functions.

### Missing ze.wrap()

Always use `ze.wrap(new OpenAI())` instead of `new OpenAI()` directly. Without wrapping, no auto-tracing occurs and prompt metadata is not extracted.

### First message is not system

The wrappers only inspect the first message for `<zeroeval>` metadata. If your first message is not a system message containing the decorated prompt, metadata linkage will not work.

### Flush before exit

For scripts, serverless functions, or short-lived processes:

```typescript
await ze.tracer.flush();
ze.tracer.shutdown();
```

Call `flush()` before `sendFeedback()` in short-lived flows to ensure spans are ingested before feedback references them.

### API URL confusion

The SDK defaults to `https://api.zeroeval.com`. Some example files use `http://localhost:8000` for local development. Do not copy localhost URLs into production code.

### Debug mode

Pass `debug: true` to `ze.init()` or set `ZEROEVAL_DEBUG=true` to see detailed span lifecycle logs.
