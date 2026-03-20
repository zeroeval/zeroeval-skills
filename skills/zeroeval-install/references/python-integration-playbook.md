# Python Integration Playbook

Complete guide for integrating ZeroEval into a Python AI application.

## Install and Initialize

### 1. Install the SDK

```bash
pip install zeroeval
```

Install with specific integrations as needed:

```bash
pip install zeroeval[openai]      # OpenAI integration
pip install zeroeval[gemini]      # Google Gemini integration
pip install zeroeval[langchain]   # LangChain integration
pip install zeroeval[langgraph]   # LangGraph integration
pip install zeroeval[all]         # All integrations
```

The SDK automatically detects and instruments installed integrations at runtime.

### 2. Get Your API Key

If you don't have an API key yet, generate one from the ZeroEval dashboard:

1. Go to [app.zeroeval.com/settings?section=project-api-keys](https://app.zeroeval.com/settings?section=project-api-keys).
2. Click **New Key**.
3. Enter a name (e.g. "dev-laptop") and pick an expiration (or "Never expires" for local dev).
4. Click **Create Key**.
5. Copy the key immediately -- it's only shown once.

Keys are scoped to a single project. If you work across multiple projects, generate a key for each one.

### 3. Configure the API Key

**Option A — Interactive CLI setup (recommended for first time):**

```bash
zeroeval setup
```

This saves `ZEROEVAL_API_KEY` to the shell configuration file (`~/.zshrc`, `~/.bashrc`). Also add it to a `.env` file in the project root for portability.

**Option B — Environment variable:**

```bash
export ZEROEVAL_API_KEY=sk_ze_your_key_here
```

**Option C — In code (least recommended for production):**

```python
import zeroeval as ze
ze.init(api_key="sk_ze_your_key_here")
```

### 4. Initialize the SDK

```python
import zeroeval as ze

ze.init()
```

`ze.init()` accepts these optional parameters:

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `api_key` | `str` | `ZEROEVAL_API_KEY` env var | API authentication |
| `api_url` | `str` | `https://api.zeroeval.com` | API endpoint (override for local dev only) |
| `organization_name` | `str` | `"Personal Organization"` | Organization display name |
| `debug` | `bool` | `False` | Verbose logging (`ZEROEVAL_DEBUG=true` also works) |
| `disabled_integrations` | `list[str]` | `None` | e.g. `["openai", "langgraph"]` |
| `enabled_integrations` | `list[str]` | `None` | Allowlist mode; disables all others |
| `sampling_rate` | `float` | `1.0` | 0.0-1.0; controls trace sampling |
| `tags` | `dict` | `None` | Global tags applied to all traces |

### 5. Initialization Order

Initialize ZeroEval **before** importing AI frameworks to ensure auto-instrumentation patches correctly:

```python
import zeroeval as ze
ze.init()

import openai  # patched automatically after ze.init()
```

If the project uses `python-dotenv`, load environment variables before `ze.init()`:

```python
from dotenv import load_dotenv
load_dotenv()

import zeroeval as ze
ze.init()
```

---

## Verify First Trace

### Automatic Tracing (OpenAI Example)

```python
import zeroeval as ze
ze.init(debug=True)

import openai

client = openai.OpenAI()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is the capital of France?"}]
)
print(response.choices[0].message.content)
```

With `debug=True`, the console output should show trace/span IDs being created and flushed. Check the ZeroEval dashboard to confirm the trace appears.

### Manual Spans

Wrap logical steps with `ze.span` for richer observability:

```python
with ze.span("user_question_answering", tags={"feature": "qa_system"}):
    with ze.span("preprocessing"):
        question = user_input.strip()

    with ze.span("llm_generation"):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": question}]
        )

    with ze.span("postprocessing"):
        answer = response.choices[0].message.content
```

### Artifact Spans

When a prompt-linked run produces multiple judged outputs, use `ze.artifact_span` instead of manually setting `completion_artifact_*` attributes:

```python
with ze.artifact_span(
    name="final-decision",
    artifact_type="final_decision",
    role="primary",
    label="Final Decision",
    tags={"judge_target": "support_ops_final_decision"},
) as s:
    s.set_io(input_data=ticket_text, output_data=decision_json)

with ze.artifact_span(
    name="customer-card",
    artifact_type="customer_card",
    role="secondary",
    label="Customer Card",
) as card:
    card.set_io(input_data=summary, output_data=decision_json)
    card.add_image(base64_data=card_image)
```

**When to use artifact spans:**
- One prompt run produces multiple outputs evaluated by different judges.
- You want the prompt completions page to show a specific output as the row preview (`role="primary"`) while keeping others accessible.
- You attach images or structured data to secondary artifacts for multimodal judge evaluation.

`artifact_span` defaults to `kind="llm"` and inherits prompt metadata automatically. It is Python SDK only for now.

**Common mistake:** Setting artifact attributes manually but forgetting `kind="llm"`. Use `ze.artifact_span` to avoid this -- it sets the correct kind by default.

### Supported Auto-Integrations

| Integration | Install Extra | Traced Operations |
|-------------|--------------|-------------------|
| OpenAI | `zeroeval[openai]` | `chat.completions.create`, `responses.create`, streaming |
| Google Gemini | `zeroeval[gemini]` | Gemini API calls |
| LangChain | `zeroeval[langchain]` | Runnables, LLMs, Tools, Retrievers, Chains |
| LangGraph | `zeroeval[langgraph]` | Graph execution, node tracing, conditional edges |

LangGraph traces include both `langgraph.*` and `langchain.*` spans (expected behavior since LangGraph builds on LangChain).

---

## ze.prompt Migration

### Why Migrate

Wrapping system prompts with `ze.prompt` enables:
- Automatic version tracking and content hashing.
- A/B testing between prompt versions.
- Automated prompt optimization from feedback data.
- Prompt-linked judge evaluations.

### Placement and Resilience

`ze.prompt()` performs a network call to register or fetch prompt versions. **Never call it at module scope or import time.** Resolve prompts inside the function or request handler where they are used.

```python
# BAD — blocks import, can timeout and crash startup
SYSTEM_PROMPT = ze.prompt(name="customer-support", content="...", from_="explicit")

# GOOD — resolves when the function runs, not at import time
DEFAULT_PROMPT = "You are a helpful customer support assistant."

def handle_request(user_message: str) -> str:
    system_prompt = ze.prompt(
        name="customer-support",
        content=DEFAULT_PROMPT,
        from_="explicit",
    )
    ...
```

For apps that must start even when the ZeroEval API is slow or unreachable, wrap the call with a fallback:

```python
DEFAULT_PROMPT = "You are a helpful customer support assistant."

def _get_system_prompt() -> str:
    try:
        return ze.prompt(
            name="customer-support",
            content=DEFAULT_PROMPT,
            from_="explicit",
        )
    except Exception:
        return DEFAULT_PROMPT

def handle_request(user_message: str) -> str:
    system_prompt = _get_system_prompt()
    ...
```

### Staged Rollout

**Stage 1 — Explicit mode (safe rollout — always returns your local content):**

```python
def handle_request(user_message: str) -> str:
    system_prompt = ze.prompt(
        name="customer-support",
        content="You are a helpful customer support assistant.",
        from_="explicit",
    )
    ...
```

This registers the prompt version in ZeroEval without swapping content at runtime. Use this to validate that metadata flows correctly. Note: `explicit` still makes a network call to register the version — it is safe from content swapping, not from network latency.

**Stage 2 — Auto mode (default, recommended for production):**

```python
def handle_request(user_message: str) -> str:
    system_prompt = ze.prompt(
        name="customer-support",
        content="You are a helpful customer support assistant.",
    )
    ...
```

With only `content` provided, the SDK tries to fetch the latest optimized version from the backend. If none exists, it falls back to the provided content. This is the recommended production mode.

**Stage 3 — Latest mode (strict, requires published version):**

```python
def handle_request(user_message: str) -> str:
    system_prompt = ze.prompt(
        name="customer-support",
        from_="latest",
    )
    ...
```

Fails if no prompt version exists in the backend. Use after the prompt library has been populated and optimized.

### Template Variables

Use `{{variable}}` syntax for dynamic prompt content:

```python
def handle_request(user_message: str, role: str, domain: str) -> str:
    system_prompt = ze.prompt(
        name="support-agent",
        content="You are a {{role}} assistant specializing in {{domain}}.",
        variables={"role": role, "domain": domain},
    )
    ...
```

Variables are interpolated by the SDK integration before the prompt reaches the LLM.

### Hash-Pinned Versions

Pin to a specific prompt version by its 64-character SHA-256 hash:

```python
def handle_request(user_message: str) -> str:
    system_prompt = ze.prompt(
        name="customer-support",
        from_="a1b2c3d4...",  # full 64-char hash
    )
    ...
```

### Prompt Metadata

`ze.prompt` attaches the metadata ZeroEval needs to version, trace, and optimize prompts. Supported integrations (OpenAI, LangChain, etc.) handle this automatically. For unsupported providers, use manual spans to link prompts to traces.

---

## Sending Feedback

After `ze.prompt` is wired, collect feedback to power optimization:

```python
feedback = ze.send_feedback(
    prompt_slug="customer-support",
    completion_id="completion-uuid-here",
    thumbs_up=True,
    reason="Response was accurate and helpful",
    expected_output="Optional ideal response text"
)
```

For scored judge feedback:

```python
feedback = ze.send_feedback(
    prompt_slug="quality-scorer",
    completion_id="completion-uuid-here",
    thumbs_up=False,
    judge_id="automation-uuid-here",
    expected_score=3.5,
    score_direction="too_high"
)
```

The `expected_output` field is used by prompt optimization to generate improved prompts.

---

## Common Pitfalls

### ze.prompt at module scope
`ze.prompt()` performs network I/O. Calling it at module level (e.g. as a global variable) blocks import and can timeout or crash startup. Always resolve prompts inside the function or request handler where they are used. See "Placement and Resilience" above.

### Import ordering
Always call `ze.init()` before importing `openai`, `langchain`, etc. Otherwise auto-instrumentation patches are not applied.

### API URL confusion
The SDK defaults to `https://api.zeroeval.com` (production). Some example files use `http://localhost:8000` for local development. Do not copy localhost URLs into production code.

### Flush in short-lived scripts
The span writer is async and buffered. For CLI scripts or one-shot jobs, ensure spans are flushed before exit:

```python
import zeroeval as ze
ze.tracer.shutdown()
```

### Debug mode
Enable `debug=True` in `ze.init()` or set `ZEROEVAL_DEBUG=true` to see detailed trace/span lifecycle logs. This is the fastest way to diagnose missing traces.

### LiveKit / OTLP conflicts
If using LiveKit or another OTLP-compatible library, disable the OpenAI integration to avoid double-patching:

```python
ze.init(disabled_integrations=["openai"])
```
