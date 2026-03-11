# Python Prompt Migration Playbook

Step-by-step guide for migrating hardcoded Python prompts to `ze.prompt`.

---

## Migration Pattern

### Before migration

```python
SYSTEM_PROMPT = "You are a helpful customer support assistant."

def handle_request(user_message: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_message},
        ],
    )
    return response.choices[0].message.content
```

### After migration (Stage 1 -- explicit mode)

```python
import zeroeval as ze
from openai import OpenAI

ze.init()
client = OpenAI()

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
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": _get_system_prompt()},
            {"role": "user", "content": user_message},
        ],
    )
    return response.choices[0].message.content
```

Key changes:

1. The hardcoded string is preserved as `DEFAULT_PROMPT` (the fallback).
2. `ze.prompt()` is called inside a helper function, not at module scope.
3. `from_="explicit"` ensures the local content is always used, while still registering the prompt version in ZeroEval.
4. A try/except falls back to the local string if ZeroEval is unreachable.

---

## Placement and Resilience

`ze.prompt()` is synchronous in Python but still performs a network call to register or fetch prompt versions.

```python
# BAD -- blocks import, can timeout and crash startup
SYSTEM_PROMPT = ze.prompt(name="my-agent", content="...", from_="explicit")

# GOOD -- resolves when the function runs, not at import time
DEFAULT_PROMPT = "You are a helpful assistant."

def _get_system_prompt() -> str:
    try:
        return ze.prompt(
            name="my-agent",
            content=DEFAULT_PROMPT,
            from_="explicit",
        )
    except Exception:
        return DEFAULT_PROMPT
```

Always call `ze.init()` before importing `openai`, `langchain`, or other provider SDKs so that auto-instrumentation patches are applied.

---

## Staged Rollout

### Stage 1 -- Explicit mode (safe rollout)

```python
system_prompt = ze.prompt(
    name="customer-support",
    content=DEFAULT_PROMPT,
    from_="explicit",
)
```

Always returns your local content. Registers the version in ZeroEval for tracking. Use this to validate that metadata flows correctly in traces before allowing the backend to swap content.

`from_="explicit"` is safe from content swapping but still makes a network call. It does not eliminate timeout risk.

### Stage 2 -- Auto mode (recommended production)

```python
system_prompt = ze.prompt(
    name="customer-support",
    content=DEFAULT_PROMPT,
)
```

The SDK tries to fetch the latest optimized version from the backend. If none exists (or on 404), it falls back to the provided content. This is the recommended steady state for production.

### Stage 3 -- Latest mode (strict)

```python
system_prompt = ze.prompt(
    name="customer-support",
    from_="latest",
)
```

Fails if no prompt version exists in the backend. Use only after the prompt library has been populated and the optimization loop is trusted.

### Hash pinning (deterministic rollback)

```python
system_prompt = ze.prompt(
    name="customer-support",
    from_="a1b2c3d4e5f6...",  # full 64-char SHA-256 hash
)
```

Fetches an exact version by content hash. Useful for rollback or controlled experiments.

---

## Template Variables

Use `{{variable}}` syntax for dynamic prompt content:

```python
system_prompt = ze.prompt(
    name="support-agent",
    content="You are a {{role}} assistant specializing in {{domain}}.",
    variables={"role": "billing", "domain": "subscription management"},
)
```

Variables are interpolated by the SDK integration before the prompt reaches the LLM.

---

## Wiring Feedback

Feedback is required for prompt optimization. Wire it as part of migration, not as a later task.

### Basic prompt feedback

```python
ze.send_feedback(
    prompt_slug="customer-support",
    completion_id="span-uuid-here",
    thumbs_up=False,
    reason="Response was too verbose",
    expected_output="A concise 2-3 sentence response",
)
```

### Getting the completion ID

The `completion_id` is the UUID of the traced span. You can obtain it from:

- The ZeroEval dashboard (inspect a trace and copy the span ID).
- The OpenAI response object's `id` field (when using auto-instrumented integrations).
- `ze.get_current_span()` if you need it programmatically within the same execution context.

### Scored judge feedback

For judges that use numeric rubrics:

```python
ze.send_feedback(
    prompt_slug="quality-scorer",
    completion_id="span-uuid-here",
    thumbs_up=False,
    judge_id="automation-uuid-here",
    expected_score=3.5,
    score_direction="too_high",
    reason="Score should have been lower due to grammar issues",
)
```

### Criterion-level judge feedback (Python only)

For scored judges with multiple criteria, Python supports per-criterion corrections:

```python
ze.send_feedback(
    prompt_slug="quality-scorer",
    completion_id="span-uuid-here",
    thumbs_up=False,
    judge_id="automation-uuid-here",
    reason="Criterion-level score adjustments",
    criteria_feedback={
        "accuracy": {"expected_score": 4.0, "reason": "Mostly correct"},
        "tone": {"expected_score": 1.0, "reason": "Too formal"},
    },
)
```

Discover available criteria first:

```python
criteria = ze.get_judge_criteria(
    project_id="your-project-id",
    judge_id="automation-uuid-here",
)
```

### Flush before feedback in short-lived scripts

The span writer is async and buffered. In CLI scripts or one-shot jobs, flush before sending feedback:

```python
ze.tracer.shutdown()
```

This ensures the span exists server-side before feedback references it.

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
3. When enough feedback exists, trigger optimization from the dashboard ("Optimize Prompt") or via CLI (`zeroeval optimize prompt start`).
4. The optimizer generates candidate prompts using the feedback data.
5. Review candidates and adopt the best one. Adoption creates a new prompt version, pins the `production` tag, and updates linked judge templates.
6. After adoption, `ze.prompt()` in auto mode automatically serves the new version.

### Optimization types

| Type | Speed | When to use |
|------|-------|-------------|
| Quick Refine | Seconds | Simple fixes, first iteration |
| Bootstrap | 1-5 minutes | Clear positive/negative signal |
| GEPA | 10-60 minutes | Complex multi-intent optimization |

---

## Validation Checklist

After migration, verify:

- [ ] `ze.init()` completes without errors.
- [ ] `ze.prompt()` returns a decorated string (contains metadata prefix).
- [ ] The traced span in the dashboard shows the prompt name and version.
- [ ] `ze.send_feedback()` returns a feedback record without errors.
- [ ] If judges are linked, evaluations appear in the judges dashboard section.

---

## Common Pitfalls

### ze.prompt at module scope

`ze.prompt()` performs network I/O. Calling it at module level blocks import and can timeout or crash startup. Always resolve prompts inside the function where they are used.

### Import ordering

Always call `ze.init()` before importing `openai`, `langchain`, etc. Otherwise auto-instrumentation patches are not applied.

### API URL confusion

The SDK defaults to `https://api.zeroeval.com` (production). Some example files use `http://localhost:8000` for local development. Do not copy localhost URLs into production code.

### Flush in short-lived scripts

The span writer is async and buffered. For CLI scripts or one-shot jobs, call `ze.tracer.shutdown()` before exit.

### Debug mode

Enable `debug=True` in `ze.init()` or set `ZEROEVAL_DEBUG=true` to see detailed trace/span lifecycle logs.
