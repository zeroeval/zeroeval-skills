# API Integration Playbook

Detailed integration guide for sending traces to ZeroEval via the REST API or OpenTelemetry (OTLP).

Base URL: `https://api.zeroeval.com`

All requests require:

```
Authorization: Bearer YOUR_ZEROEVAL_API_KEY
Content-Type: application/json
```

---

## REST API Quick Start

The fastest path: send a single span via `POST /spans`. The trace (and session, if specified) are auto-created.

### Minimal Span (cURL)

```bash
curl -X POST https://api.zeroeval.com/spans \
  -H "Authorization: Bearer $ZEROEVAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[{
    "trace_id": "'"$(uuidgen | tr '[:upper:]' '[:lower:]')"'",
    "name": "chat_completion",
    "kind": "llm",
    "started_at": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",
    "status": "ok",
    "input_data": "[{\"role\": \"user\", \"content\": \"Hello\"}]",
    "output_data": "Hi there!"
  }]'
```

### Minimal Span (Python, no SDK)

```python
import requests, json, uuid
from datetime import datetime, timezone

response = requests.post(
    "https://api.zeroeval.com/spans",
    headers={"Authorization": f"Bearer {ZEROEVAL_API_KEY}"},
    json=[{
        "trace_id": str(uuid.uuid4()),
        "name": "chat_completion",
        "kind": "llm",
        "started_at": datetime.now(timezone.utc).isoformat(),
        "status": "ok",
        "input_data": json.dumps([{"role": "user", "content": "Hello"}]),
        "output_data": "Hi there!"
    }]
)
assert response.status_code == 200, f"Failed: {response.text}"
```

### Minimal Span (Go)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "time"

    "github.com/google/uuid"
)

func main() {
    spans := []map[string]any{{
        "trace_id":   uuid.New().String(),
        "name":       "chat_completion",
        "kind":       "llm",
        "started_at": time.Now().UTC().Format(time.RFC3339),
        "status":     "ok",
        "input_data": `[{"role": "user", "content": "Hello"}]`,
        "output_data": "Hi there!",
    }}

    body, _ := json.Marshal(spans)
    req, _ := http.NewRequest("POST", "https://api.zeroeval.com/spans", bytes.NewReader(body))
    req.Header.Set("Authorization", "Bearer "+os.Getenv("ZEROEVAL_API_KEY"))
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    fmt.Println("Status:", resp.Status)
}
```

After sending, open the ZeroEval dashboard. The span should appear within seconds. If it does not, jump to the Troubleshooting section below.

---

## Adding Structure

### Sessions

Sessions group related traces (e.g. a user conversation across multiple requests). Pass a `session_id` on your spans, or use the inline `session` object to auto-create a named session:

```json
[{
  "trace_id": "...",
  "name": "turn_1",
  "started_at": "...",
  "session": {"id": "SESSION_UUID", "name": "User 42 conversation"}
}]
```

You can also pre-create sessions via `POST /sessions` if you need to set tags or attributes upfront:

```bash
curl -X POST https://api.zeroeval.com/sessions \
  -H "Authorization: Bearer $ZEROEVAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[{
    "project_id": "YOUR_PROJECT_ID",
    "name": "User 42 conversation",
    "tags": {"user_id": "42", "environment": "production"}
  }]'
```

### Nested Spans

Use `parent_span_id` to build a span tree within a single trace. All spans in a trace share the same `trace_id`.

```json
[
  {
    "id": "PARENT_SPAN_UUID",
    "trace_id": "TRACE_UUID",
    "name": "agent_run",
    "kind": "generic",
    "started_at": "2025-01-15T10:30:00Z",
    "ended_at": "2025-01-15T10:30:05Z",
    "status": "ok"
  },
  {
    "trace_id": "TRACE_UUID",
    "parent_span_id": "PARENT_SPAN_UUID",
    "name": "llm_call",
    "kind": "llm",
    "started_at": "2025-01-15T10:30:01Z",
    "ended_at": "2025-01-15T10:30:03Z",
    "status": "ok",
    "attributes": {
      "provider": "openai",
      "model": "gpt-4o",
      "inputTokens": 150,
      "outputTokens": 230
    },
    "input_data": "[{\"role\": \"user\", \"content\": \"Summarize this\"}]",
    "output_data": "Here is a summary..."
  }
]
```

Both spans can be sent in a single `POST /spans` request. The dashboard renders them as a parent-child tree.

### LLM Cost Tracking

For automatic cost calculation, set `kind` to `"llm"` and include these attributes:

| Attribute      | Required | Description                             |
| -------------- | -------- | --------------------------------------- |
| `provider`     | Yes      | `"openai"`, `"gemini"`, `"anthropic"`   |
| `model`        | Yes      | `"gpt-4o"`, `"claude-3-5-sonnet"`, etc. |
| `inputTokens`  | Yes      | Number of input tokens                  |
| `outputTokens` | Yes      | Number of output tokens                 |

Cost formula: `(inputTokens * inputPrice + outputTokens * outputPrice) / 1,000,000`

### Tags

Tags are arbitrary key-value pairs for filtering in the dashboard. Set them at the span, trace, or session level:

```json
[{
  "trace_id": "...",
  "name": "chat_completion",
  "started_at": "...",
  "tags": {"environment": "production", "feature": "search"},
  "trace_tags": {"user_id": "42"},
  "session_tags": {"plan": "pro"}
}]
```

### Error Reporting

Record errors on spans for debugging:

```json
[{
  "trace_id": "...",
  "name": "chat_completion",
  "started_at": "...",
  "status": "error",
  "error_code": "RateLimitError",
  "error_message": "Rate limit exceeded",
  "error_stack": "Traceback (most recent call last):\n  ..."
}]
```

---

## Span Field Reference

| Field            | Type            | Required | Default         | Description                                                  |
| ---------------- | --------------- | -------- | --------------- | ------------------------------------------------------------ |
| `trace_id`       | `string (UUID)` | Yes      | --              | Trace this span belongs to                                   |
| `name`           | `string`        | Yes      | --              | Descriptive name                                             |
| `started_at`     | `ISO 8601`      | Yes      | --              | When the span started                                        |
| `id`             | `string (UUID)` | No       | auto-generated  | Client-provided span ID                                      |
| `kind`           | `string`        | No       | `"generic"`     | `generic`, `llm`, `tts`, `http`, `database`, `vector_store`  |
| `status`         | `string`        | No       | `null`          | `unset`, `ok`, `error`                                       |
| `ended_at`       | `ISO 8601`      | No       | `null`          | When the span completed                                      |
| `duration_ms`    | `float`         | No       | `null`          | Duration in milliseconds                                     |
| `cost`           | `float`         | No       | auto-calculated | Cost (auto-calculated for LLM spans)                         |
| `parent_span_id` | `string (UUID)` | No       | `null`          | Parent span for nesting                                      |
| `session_id`     | `string (UUID)` | No       | `null`          | Session to associate with                                    |
| `session`        | `object`        | No       | `null`          | `{"id": "...", "name": "..."}` -- alternative to session_id  |
| `input_data`     | `string`        | No       | `null`          | Input data (JSON string for messages)                        |
| `output_data`    | `string`        | No       | `null`          | Output/response text                                         |
| `attributes`     | `object`        | No       | `{}`            | Arbitrary key-value attributes                               |
| `tags`           | `object`        | No       | `{}`            | Key-value tags for filtering                                 |
| `trace_tags`     | `object`        | No       | `{}`            | Tags applied to the parent trace                             |
| `session_tags`   | `object`        | No       | `{}`            | Tags applied to the parent session                           |
| `error_code`     | `string`        | No       | `null`          | Error code or exception class                                |
| `error_message`  | `string`        | No       | `null`          | Error description                                            |
| `error_stack`    | `string`        | No       | `null`          | Stack trace                                                  |
| `code_filepath`  | `string`        | No       | `null`          | Source file path                                             |
| `code_lineno`    | `int`           | No       | `null`          | Source line number                                           |

---

## OTLP Quick Start

If the application already uses OpenTelemetry, export traces to ZeroEval's OTLP endpoint.

### Endpoint

```
POST https://api.zeroeval.com/v1/traces
```

Headers:

```
Authorization: Bearer YOUR_ZEROEVAL_API_KEY
Content-Type: application/json  (or application/x-protobuf)
```

### Option A: OpenTelemetry Collector

Create `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

exporters:
  otlphttp:
    endpoint: https://api.zeroeval.com
    headers:
      Authorization: "Bearer ${env:ZEROEVAL_API_KEY}"
    traces_endpoint: https://api.zeroeval.com/v1/traces

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp]
```

Run with Docker:

```bash
docker run --rm \
  -v $(pwd)/otel-collector-config.yaml:/etc/config.yaml \
  -e ZEROEVAL_API_KEY=$ZEROEVAL_API_KEY \
  -p 4317:4317 -p 4318:4318 \
  otel/opentelemetry-collector-contrib:latest \
  --config=/etc/config.yaml
```

### Option B: Direct OTLP Export (Python)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint="https://api.zeroeval.com/v1/traces",
    headers={"Authorization": "Bearer YOUR_ZEROEVAL_API_KEY"}
)

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
```

### Option C: Direct OTLP Export (Node.js)

```typescript
import { NodeTracerProvider } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";

const exporter = new OTLPTraceExporter({
  url: "https://api.zeroeval.com/v1/traces",
  headers: { Authorization: "Bearer YOUR_ZEROEVAL_API_KEY" },
});

const provider = new NodeTracerProvider();
provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();
```

### OTLP Attribute Mapping

ZeroEval maps standard OTLP attributes automatically:

| OTLP Attribute          | ZeroEval Field   |
| ----------------------- | ---------------- |
| `span.name`             | `name`           |
| `span.kind`             | `kind`           |
| `span.status`           | `status`         |
| `span.start_time`       | `started_at`     |
| `span.end_time`         | `ended_at`       |
| `span.trace_id`         | `trace_id`       |
| `span.parent_span_id`   | `parent_span_id` |
| `span.attributes.*`     | `attributes`     |

For LLM cost tracking via OTLP, set these span attributes:

| Attribute                                               | Description          |
| ------------------------------------------------------- | -------------------- |
| `llm.provider` or `gen_ai.system`                       | Provider name        |
| `llm.model` or `gen_ai.request.model`                   | Model identifier     |
| `llm.input_tokens` or `gen_ai.usage.prompt_tokens`      | Input token count    |
| `llm.output_tokens` or `gen_ai.usage.completion_tokens` | Output token count   |

For session grouping via OTLP, set:

| Attribute               | Description                    |
| ----------------------- | ------------------------------ |
| `zeroeval.session.id`   | Session ID for grouping traces |
| `zeroeval.session.name` | Human-readable session name    |

---

## Troubleshooting

### No Spans Visible in Dashboard

```
Is the API key valid?
├── No (401/403 response) -> Regenerate at app.zeroeval.com/settings?section=project-api-keys
└── Yes
    Is the request hitting the correct URL?
    ├── Using localhost -> Switch to https://api.zeroeval.com for production
    └── Using production URL
        Is the response 200?
        ├── No (400) -> Check payload: trace_id, name, and started_at are required
        ├── No (5xx) -> Backend issue. Retry or contact founders@zeroeval.com
        └── Yes -> Span should appear in dashboard within seconds. Try refreshing.
```

### Common Payload Mistakes

| Symptom | Cause | Fix |
|---------|-------|-----|
| 400 Bad Request | Missing `trace_id`, `name`, or `started_at` | Include all three required fields |
| 400 Bad Request | `trace_id` is not a valid UUID | Use a proper UUID v4 string |
| 400 Bad Request | `started_at` is not ISO 8601 | Use format like `2025-01-15T10:30:00Z` |
| Cost not calculated | Missing `kind: "llm"` or token attributes | Set `kind` and include `provider`, `model`, `inputTokens`, `outputTokens` in attributes |
| Spans not grouped | Different `trace_id` values | Use the same `trace_id` for all spans in one trace |

### OTLP-Specific Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Collector not exporting | Wrong endpoint URL | Use `https://api.zeroeval.com/v1/traces` (not `/spans`) |
| Auth failure from collector | Missing or wrong header | Set `Authorization: "Bearer ${env:ZEROEVAL_API_KEY}"` in exporter headers |
| Spans arrive but no cost | Missing LLM attributes | Set `llm.provider`, `llm.model`, `llm.input_tokens`, `llm.output_tokens` on spans |

---

## Getting Help

- Email: founders@zeroeval.com
- Docs: https://docs.zeroeval.com
- API reference: https://docs.zeroeval.com/tracing/api-reference
- OpenTelemetry guide: https://docs.zeroeval.com/tracing/opentelemetry
