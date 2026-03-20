---
name: zeroeval-install
description: This skill should be used when users want to install, set up, or integrate ZeroEval into their AI application, agent, or pipeline. It covers SDK setup (Python and TypeScript), first-run tracing, ze.prompt migration, and judge recommendations. For non-SDK languages or direct API/OTLP ingestion it routes to the custom-tracing skill. Triggers on "install zeroeval", "set up zeroeval", "add tracing", "integrate zeroeval", "ze.prompt", "add judges", or "monitor my AI app".
---

# ZeroEval Install and Integrate

Guide users from zero to production-ready ZeroEval integration: tracing, prompt management, and automated judges.

## When To Use

- Setting up ZeroEval for the first time in any language.
- Adding tracing/observability to an existing AI app, agent, or pipeline.
- Migrating hardcoded prompts to `ze.prompt` with staged rollout (Python / TypeScript).
- Choosing and configuring judges for automated evaluation.
- Troubleshooting missing traces, broken feedback loops, or prompt metadata issues.

## Execution Sequence

Follow these steps in order. Each step references a specific playbook in `references/` for deep details; load only the relevant playbook when needed.

### Step 1: Detect Integration Path

Determine which integration path fits the user's setup:

- Check for `pyproject.toml`, `requirements.txt`, `setup.py`, or `.py` files -> **Python SDK path**. Continue to Step 2.
- Check for `package.json`, `tsconfig.json`, or `.ts`/`.js` files -> **TypeScript SDK path**. Continue to Step 2.
- If the user's language has no ZeroEval SDK (Go, Ruby, Java, Rust, etc.), or they explicitly want to use the REST API or OpenTelemetry without an SDK -> **Direct API / OTLP path**. Hand off to the `custom-tracing` skill and stop here.
- If both Python and TypeScript are present, ask the user which SDK to set up first.

### Step 2: Install and Initialize

Load the appropriate playbook:

- **Python**: Read `references/python-integration-playbook.md` and follow the "Install and Initialize" section.
- **TypeScript**: Read `references/typescript-integration-playbook.md` and follow the "Install and Initialize" section.

Minimum outcome: `ze.init()` runs without errors and the API key is configured.

### Step 3: Verify First Trace

Make one LLM call through a supported integration and confirm a trace appears.

- **Python**: Follow the "Verify First Trace" section of the Python playbook. If the user's agent produces multiple judged outputs per run, introduce `ze.artifact_span` (see "Artifact Spans" in the playbook).
- **TypeScript**: Follow the "Verify First Trace" section of the TypeScript playbook.

Minimum outcome: at least one span is ingested (confirm via dashboard or debug logs).

### Step 4: Suggest ze.prompt Migration

If the user has hardcoded system prompts, propose migrating to `ze.prompt` for version tracking, A/B testing, and prompt optimization.

- Follow the "ze.prompt Migration" section of the relevant SDK playbook.
- Start with `from: "explicit"` (safe rollout mode — always returns your local content, but still registers the version via a network call), then graduate to auto mode.
- **Always place `ze.prompt()` inside the function or request handler where the prompt is used.** It performs network I/O and must not run at module import time or during app startup. See the playbook's "Placement and Resilience" guidance.

For the full migration workflow including feedback wiring, judge linkage, staged rollout, and prompt optimization, use the `prompt-migration` skill.

### Step 5: Suggest Judges

Load `references/judges-playbook.md` and recommend starter judges based on the user's app pattern:

- Customer support / chat agents
- Extraction / classification pipelines
- Coding copilots
- Retrieval QA / RAG assistants

Minimum outcome: user understands binary vs scored judges and has a first judge created or planned.

### Step 6: Validate and Troubleshoot

Run the final checklist. If any check fails, load `references/troubleshooting.md` for diagnostics.

- [ ] `ze.init()` completes without errors
- [ ] At least one trace is visible in the dashboard (or debug logs confirm span flush)
- [ ] `ze.prompt` returns decorated content with prompt metadata (if adopted)
- [ ] Feedback or judge evaluation path is wired (if judges are configured)

## Key Principles

- **Minimal first**: get one trace working before introducing prompts or judges.
- **Staged rollout**: always start `ze.prompt` with `from: "explicit"`, then auto, then `from: "latest"`.
- **Lazy prompt resolution**: call `ze.prompt()` inside the function or request path where the prompt is used, never at module scope or import time. It performs network I/O and can block or timeout during startup.
- **Evidence over assumption**: use `debug: true` / `ZEROEVAL_DEBUG=true` to confirm SDK behavior rather than guessing.
- **Cloud by default**: the production API URL is `https://api.zeroeval.com`. Only use `http://localhost:8000` for local development with an explicit override.
