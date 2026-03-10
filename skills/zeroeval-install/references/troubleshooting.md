# Troubleshooting

Fast diagnostics for common onboarding issues.

---

## Diagnostic Decision Tree

### No Traces Visible

```
Is ZEROEVAL_API_KEY set?
├── No  -> Generate one: app.zeroeval.com/settings?section=project-api-keys > New Key
│         Then: `zeroeval setup` (Python) or `export ZEROEVAL_API_KEY=...`
└── Yes
    Is ze.init() being called?
    ├── No  -> Add ze.init() before any LLM calls
    └── Yes
        Is debug mode on?
        ├── No  -> Enable: ze.init(debug=True) or ZEROEVAL_DEBUG=true
        └── Yes
            Are span flush logs appearing in console?
            ├── No  -> Check: is the script exiting before flush?
            │         Fix: add tracer.shutdown() (Python) or ze.tracer.shutdown() (TS)
            └── Yes
                Is API URL pointing to production?
                ├── No  -> Change api_url to https://api.zeroeval.com
                └── Yes
                    Check: are there HTTP errors in debug output?
                    ├── 401/403 -> API key is invalid or expired. Regenerate in dashboard.
                    └── 5xx     -> Backend issue. Retry or contact founders@zeroeval.com.
```

### Traces Visible but No Auto-Instrumentation

**Python:**
- Confirm `ze.init()` is called **before** importing the AI framework (`openai`, `langchain`, etc.).
- Verify the integration is not disabled: check `disabled_integrations` param and `ZEROEVAL_DISABLED_INTEGRATIONS` env var.
- Verify the integration package is installed: `pip list | grep openai`.
- Enable debug mode (`ze.init(debug=True)`) and check the console for which integrations are active.

**TypeScript:**
- Confirm the OpenAI client is wrapped: `const openai = ze.wrap(new OpenAI())`.
- `ze.wrap()` only supports OpenAI and Vercel AI SDK. For other providers, use manual spans.
- Check `integrations` option in `ze.init()` for disabled entries.

### App Hangs or Crashes on Startup (ze.prompt Timeout)

**Symptoms:** `ReadTimeout` from `api.zeroeval.com` during import, app never starts, or agent file fails to load.

```
Is ze.prompt() called at module scope or import time?
├── Yes -> Move it into the function or request handler where the prompt is used.
│         ze.prompt() performs network I/O and must not run during module evaluation.
│         See "Placement and Resilience" in the relevant SDK playbook.
└── No
    Is the network or API reachable?
    ├── No  -> Check connectivity, firewall, or proxy settings.
    │         For startup resilience, wrap ze.prompt() in try/except (Python) or
    │         try/catch (TS) and fall back to the local prompt string.
    └── Yes
        Is the timeout unusually short?
        └── Default is 10s. If still timing out, check for slow DNS or proxy overhead.
            Consider the try/except fallback pattern to keep the app running.
```

**Key point:** `from_="explicit"` / `from: "explicit"` avoids backend content swapping but still makes a network call to register the prompt version. It does not eliminate the timeout risk.

### ze.prompt Metadata Not Linking

- Confirm the prompt is passed as the **system message** (first message with `role: "system"`).
- Supported integrations (OpenAI, Vercel AI) handle prompt metadata automatically.
- If using an unsupported provider, use manual spans instead of relying on automatic metadata handling.
- Enable debug mode to confirm prompt version and task ID are being picked up.

### Feedback Calls Failing

- Ensure spans are flushed **before** calling `send_feedback` / `sendFeedback`. The backend must have the span ingested before feedback can reference it.
  ```python
  # Python
  import zeroeval as ze
  ze.tracer.flush()
  ze.send_feedback(...)
  ```
  ```typescript
  // TypeScript
  await ze.tracer.flush();
  await ze.sendFeedback({ ... });
  ```
- Verify `completion_id` / `completionId` is a valid span UUID (not a response ID from the LLM provider).
- Check that `prompt_slug` / `promptSlug` matches the `name` used in `ze.prompt`.

### Local Dev vs Production URL Confusion

| Context | API URL |
|---------|---------|
| Production (default) | `https://api.zeroeval.com` |
| Local development | `http://localhost:8000` |

The SDK defaults to production. Only override `api_url` / `apiUrl` when running the backend locally. Never commit `localhost:8000` to production config.

---

## 10-Minute Quickstart Route

For users who want the fastest path to value:

### Python (5 minutes)

First, grab an API key: [app.zeroeval.com/settings?section=project-api-keys](https://app.zeroeval.com/settings?section=project-api-keys) > **New Key**. Copy it.

```bash
pip install zeroeval[openai]
zeroeval setup          # paste your key when prompted
```

```python
import zeroeval as ze
ze.init()

import openai
client = openai.OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.choices[0].message.content)
```

Open the ZeroEval dashboard. The trace should appear within seconds.

### TypeScript (5 minutes)

First, grab an API key: [app.zeroeval.com/settings?section=project-api-keys](https://app.zeroeval.com/settings?section=project-api-keys) > **New Key**. Copy it.

```bash
npm install zeroeval openai
export ZEROEVAL_API_KEY=sk_ze_your_key    # paste your key here
export OPENAI_API_KEY=sk-your_key
```

```typescript
import * as ze from "zeroeval";
import { OpenAI } from "openai";

ze.init();
const openai = ze.wrap(new OpenAI());

const res = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "Hello!" }],
});
console.log(res.choices[0].message.content);
ze.tracer.shutdown();
```

Open the ZeroEval dashboard. The trace should appear within seconds.

---

## Recommended Progression Path

After the quickstart, here is the recommended order for deepening the integration:

| Stage | What To Do | Time Estimate |
|-------|-----------|---------------|
| 1. Basic tracing | `ze.init()` + auto-instrumentation | 5 min |
| 2. Manual spans | Wrap logical pipeline steps with `ze.span` / `withSpan` | 15 min |
| 3. ze.prompt (explicit) | Replace hardcoded system prompts with `ze.prompt(..., from_="explicit")` inside request/function flow | 20 min |
| 4. ze.prompt (auto) | Switch to auto mode for production prompt optimization (still inside request flow) | 5 min |
| 5. First judge | Create a binary judge for the most critical quality check | 10 min |
| 6. Feedback loop | Wire `send_feedback` for human-in-the-loop corrections | 15 min |
| 7. Scored judges | Add scored judges with rubric criteria for nuanced evaluation | 15 min |
| 8. Prompt optimization | Use accumulated feedback data for prompt optimization | Dashboard-driven |

---

## What To Do Next After Success

Once traces, prompts, and judges are wired:

- **Monitor the judges digest** in the dashboard to track positive rates and coverage gaps.
- **Review flagged evaluations** where judge confidence is low or human feedback disagrees.
- **Run prompt optimization** when sufficient feedback has accumulated (visible in the Prompt Library).
- **Add A/B tests** using `ze.choose()` (Python) to compare prompt variants with real traffic.
- **Set sampling rates** for high-volume production apps to control cost: `ze.init(sampling_rate=0.1)`.
- **Tag traces** with environment, user, or feature metadata for filtering in the dashboard.

---

## Getting Help

- Email: founders@zeroeval.com
- Docs: https://docs.zeroeval.com
- Debug mode: always the first step for diagnosing issues (`debug=True` / `ZEROEVAL_DEBUG=true`)
