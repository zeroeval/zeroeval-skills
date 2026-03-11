---
name: prompt-migration
description: This skill should be used when users want to migrate hardcoded prompts to ze.prompt for version tracking, feedback collection, judge linkage, and prompt optimization. It covers the full migration workflow for both Python and TypeScript. Triggers on "migrate prompt", "ze.prompt", "hardcoded prompt", "prompt migration", "send feedback", "prompt optimization", "wire feedback", or "connect judges to prompts".
---

# Prompt Migration

Guide users through replacing hardcoded LLM prompts with `ze.prompt`, wiring feedback, connecting judges, and enabling prompt optimization.

## When To Use

- Migrating one or more hardcoded system prompts to `ze.prompt`.
- Wiring feedback collection (`send_feedback` / `sendFeedback`) after migration.
- Connecting judges to a migrated prompt for automated evaluation and optimization.
- Understanding the staged rollout from `explicit` to auto to `latest` mode.
- Troubleshooting prompt metadata, trace linkage, or feedback delivery after migration.
- Graduating from `ze.prompt` registration to full prompt optimization.

## Prerequisites

The user must have ZeroEval installed and tracing working before migrating prompts. If `ze.init()` is not yet configured, use the `zeroeval-install` skill first.

## Execution Sequence

Follow these steps in order. Load reference playbooks only when needed for the current step.

### Step 1: Detect Stack and Provider

Determine the user's language and LLM provider integration.

- Check for `.py` files, `pyproject.toml`, or `requirements.txt` -> **Python path**.
- Check for `.ts`/`.js` files, `package.json`, or `tsconfig.json` -> **TypeScript path**.
- Identify the LLM provider: OpenAI, Vercel AI, LangChain, LangGraph, or unsupported.

### Step 2: Locate the Hardcoded Prompt

Search the codebase for the system prompt string that will be migrated. Common patterns:

- A string constant at module scope used as the system message.
- A prompt embedded inline in the LLM call.
- A prompt loaded from a file or config.

Identify the exact callsite where the prompt is passed to the LLM. Migration replaces only that callsite.

### Step 3: Apply the Safe Migration Pattern

Load the appropriate playbook and follow the migration section:

- **Python**: Read `references/python-prompt-migration-playbook.md`.
- **TypeScript**: Read `references/typescript-prompt-migration-playbook.md`.

Core rules for both languages:

1. Keep the existing hardcoded prompt as a local fallback constant.
2. Resolve `ze.prompt()` inside the function or request handler where it is used. It performs network I/O and must not run at module scope, import time, or top-level `await`.
3. Start with `explicit` mode (safe rollout -- always returns your local content, but registers the version via a network call).
4. Wrap with try/except (Python) or try/catch (TypeScript) to fall back to the local constant if ZeroEval is unreachable.

### Step 4: Validate Trace Metadata

After migrating, make one LLM call and confirm prompt metadata is linked in traces.

- **Dashboard**: open the trace and verify the span shows the prompt name, version, and content hash.
- **Debug logs**: enable `debug=True` / `debug: true` in `ze.init()` and look for prompt registration logs.
- **First system message rule (TypeScript)**: the decorated prompt must be the first `system` message. If it is not, metadata extraction will not happen automatically.

### Step 5: Wire Feedback Collection

Feedback is what powers prompt optimization. Follow the feedback section of the relevant playbook.

Key points:

- Use the traced span ID as the `completion_id` / `completionId`.
- In short-lived scripts or serverless functions, flush spans before sending feedback so the span exists server-side first.
- Include `reason` and `expected_output` / `expectedOutput` when possible -- these give the optimizer concrete signal.

### Step 6: Connect Judges

If the user has judges (or should create them), recommend linking judges to the migrated prompt.

- Prompt-linked judges auto-evaluate spans from that prompt and feed results into optimization.
- Linking is done via the dashboard or `POST /signals/projects/{project_id}/prompts/{prompt_id}/judges/{automation_id}/link`.
- If no judges exist yet, suggest using the `create-judge` skill to design and create one.

### Step 7: Graduate Rollout Mode

Once metadata is flowing and feedback is wired, graduate from `explicit` to production modes:

1. **Auto mode** (recommended steady state): remove the `from_` / `from` parameter and provide only `content`. The SDK fetches the latest optimized version if one exists, otherwise uses your local content.
2. **Latest mode** (strict): use `from_="latest"` / `from: "latest"` when you want to require a published version from the backend.
3. **Hash pinning** (deterministic): use `from_="<64-char-hash>"` / `from: "<64-char-hash>"` to pin to a specific version for rollback or controlled experiments.

### Step 8: Troubleshoot

If any step fails, load the relevant playbook's troubleshooting section. Common issues:

- Prompt metadata not appearing in traces -> check provider wrapper and first-system-message rule.
- Feedback returns 404 -> flush spans before sending feedback; the span must be ingested first.
- `ze.prompt()` timing out -> move out of module scope; add try/except fallback.
- Optimization not starting -> need sufficient feedback volume and linked judges.

## Key Principles

- **One prompt at a time**: migrate and validate a single prompt before moving to the next.
- **Explicit first, auto later**: always start with `from: "explicit"` to validate metadata flow before enabling auto-optimization.
- **Feedback is not optional**: prompt optimization requires feedback. Wire it as part of migration, not as a later task.
- **Judges close the loop**: linking a judge to a prompt turns judge evaluations into optimization signal.
- **Lazy resolution**: `ze.prompt()` performs network I/O. Resolve it inside the execution path, never at module scope or startup.
- **Local fallback**: always keep the hardcoded prompt available as a fallback for resilience.
