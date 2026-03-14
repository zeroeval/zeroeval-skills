---
name: custom-tracing
description: This skill should be used when users want to send traces to ZeroEval without installing the SDK, using the REST API or OpenTelemetry (OTLP) directly. It covers direct HTTP span ingestion, OTLP collector configuration, and first-trace verification for any language. Triggers on "send traces via API", "direct API tracing", "custom tracing", "manual tracing", "without SDK", "unsupported language", "REST API tracing", "OTLP", "OpenTelemetry", or language cues like "Go", "Ruby", "Java", "Rust", "Elixir", or "PHP".
---

# Custom Tracing (Direct API / OTLP)

Guide users through sending traces to ZeroEval without the Python or TypeScript SDK, using the REST API or OpenTelemetry protocol.

## When To Use

- The user's language or runtime has no ZeroEval SDK (Go, Ruby, Java, Rust, Elixir, PHP, etc.).
- The user wants to send spans over plain HTTP from any environment.
- The user already has OpenTelemetry instrumentation and wants to export to ZeroEval.
- The user prefers a vendor-neutral or SDK-free integration path.
- The user explicitly asks about `POST /spans`, the REST API, or OTLP ingestion.

Do **not** use this skill when the user is working in Python or TypeScript and wants the full SDK experience (auto-instrumentation, `ze.prompt`, etc.). Use `zeroeval-install` instead.

## Prerequisites

- A ZeroEval account and API key from [Settings -> API Keys](https://app.zeroeval.com/settings?section=api-keys).
- An HTTP client or OpenTelemetry exporter in the user's language of choice.

## Execution Sequence

Follow these steps in order. Load the reference playbook only when needed for detailed payloads and examples.

### Step 1: Choose Integration Path

Ask the user which path fits their setup:

- **REST API** -- send spans directly via `POST /spans`. Best for custom integrations, scripts, or languages without OpenTelemetry support.
- **OpenTelemetry (OTLP)** -- export traces via the standard OTLP protocol to `POST /v1/traces`. Best when the app already uses OpenTelemetry or needs multi-backend fan-out.

If the user is unsure, default to REST API -- it has fewer dependencies and works from any language with an HTTP client.

### Step 2: Configure Authentication

The API key is passed as a Bearer token in every request:

```
Authorization: Bearer YOUR_ZEROEVAL_API_KEY
```

Recommend storing the key in an environment variable (`ZEROEVAL_API_KEY`) rather than hardcoding it.

### Step 3: Send First Trace

Load `references/api-integration-playbook.md` and follow the section matching the chosen path:

- **REST API**: Follow the "REST API Quick Start" section.
- **OTLP**: Follow the "OTLP Quick Start" section.

Minimum outcome: at least one span is ingested and visible in the ZeroEval dashboard.

### Step 4: Add Structure (Sessions and Nested Spans)

Once the first trace is confirmed, guide the user through optional structure:

- **Sessions**: group related traces by passing `session_id` or the `session` object on spans.
- **Nested spans**: use `parent_span_id` to build a tree of operations within a trace.
- **LLM cost tracking**: set `kind: "llm"` and include `provider`, `model`, `inputTokens`, `outputTokens` in attributes for automatic cost calculation.

Details are in the "Adding Structure" section of the playbook.

### Step 5: Validate and Troubleshoot

Run the checklist:

- [ ] API key is valid (no 401/403 responses)
- [ ] At least one span is visible in the dashboard
- [ ] `trace_id` groups spans into a single trace view
- [ ] Session grouping works (if configured)
- [ ] LLM cost appears on spans with `kind: "llm"` and token attributes

If any check fails, follow the "Troubleshooting" section of the playbook.

### Step 6: Suggest Next Steps

After tracing is working:

- **Judges**: recommend the `create-judge` skill to set up automated evaluation on ingested traces.
- **Feedback**: point users to the Feedback API (`POST /feedback`) for human-in-the-loop review.
- **SDK upgrade**: if the user later adopts Python or TypeScript, suggest `zeroeval-install` for auto-instrumentation and `ze.prompt` support.

## Key Principles

- **Spans are the entry point**: `POST /spans` auto-creates traces and sessions. Start there, not with `POST /traces` or `POST /sessions`.
- **One span, one trace**: the simplest integration is a single span per LLM call. Add nesting only when the user needs it.
- **Cloud by default**: the production base URL is `https://api.zeroeval.com`. Only use `http://localhost:8000` for local development.
- **Evidence over assumption**: confirm spans appear in the dashboard before adding complexity.
