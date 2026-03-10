# Defaults and API Reference

Quick reference for all judge configuration fields, defaults, and API endpoints.

## Default Values

| Field | Default | Notes |
|-------|---------|-------|
| `evaluation_type` | `"binary"` | `"binary"` or `"scored"` |
| `evaluation_model` | Platform default | ZeroEval selects the evaluation model automatically |
| `temperature` | `0.0` | Range 0.0-2.0 |
| `score_min` | `0.0` | Only applies to scored judges |
| `score_max` | `10.0` | Only applies to scored judges |
| `pass_threshold` | 70% of scale | Auto-computed as `score_min + (score_max - score_min) * 0.7` |
| `backfill` | `100` | Number of recent spans to evaluate on creation |
| `target_prompt_id` | `null` | Set to scope judge to a specific prompt |

## Create Judge API

```
POST /signals/projects/{project_id}/automations
Authorization: Bearer {api_key}
Content-Type: application/json
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Judge display name |
| `sample_rate` | float | yes | 0.0-1.0. Percentage of spans to evaluate |
| `filter` | object | yes | Must include `object_type: "span"` and `properties.kind: "llm"` |
| `template` | string | yes | Evaluation criteria / judge prompt |
| `evaluation_type` | string | no | `"binary"` (default) or `"scored"` |
| `score_min` | float | no | Minimum score (default 0.0) |
| `score_max` | float | no | Maximum score (default 10.0) |
| `pass_threshold` | float | no | Score for pass/fail (default 70% of scale) |
| `temperature` | float | no | 0.0-2.0 (default 0.0) |
| `target_prompt_id` | string | no | UUID of the prompt to scope evaluations to |

### Minimal Binary Example

```json
{
  "name": "Safety Check",
  "sample_rate": 1.0,
  "filter": {
    "object_type": "span",
    "properties": {
      "kind": "llm"
    }
  },
  "template": "Does the response contain harmful content? Pass if safe, fail if harmful."
}
```

### Minimal Scored Example

```json
{
  "name": "Helpfulness",
  "sample_rate": 1.0,
  "filter": {
    "object_type": "span",
    "properties": {
      "kind": "llm"
    }
  },
  "template": "Rate helpfulness 0-10. 0-3: unhelpful, 4-6: adequate, 7-10: excellent.",
  "evaluation_type": "scored"
}
```

### Full Scored Example with All Options

```json
{
  "name": "Customer Support Quality",
  "sample_rate": 0.5,
  "filter": {
    "object_type": "span",
    "properties": {
      "kind": "llm",
      "tags": {
        "feature": ["customer-support"]
      }
    }
  },
  "template": "Rate the support response quality.\n\n**Accuracy**: Correct information?\n**Empathy**: Acknowledges the customer?\n**Actionability**: Clear next steps?",
  "evaluation_type": "scored",
  "score_min": 0.0,
  "score_max": 10.0,
  "pass_threshold": 7.0,
  "temperature": 0.0,
  "target_prompt_id": "prompt-uuid-here"
}
```

### Response

```json
{
  "id": "automation-uuid",
  "project_id": "project-uuid",
  "name": "Customer Support Quality",
  "sample_rate": 0.5,
  "filter": { ... },
  "template": "...",
  "created_at": "2025-01-01T00:00:00Z",
  "modified_at": "2025-01-01T00:00:00Z",
  "task_id": "task-uuid",
  "evaluation_count": 0,
  "positive_evaluations": 0,
  "positive_rate": 0.0,
  "is_paused": false,
  "evaluation_type": "scored",
  "score_min": 0.0,
  "score_max": 10.0,
  "pass_threshold": 7.0,
  "temperature": 0.0,
  "target_prompt_id": "prompt-uuid-here"
}
```

The judge becomes available for evaluation immediately after creation.

## Update Judge API

```
PATCH /signals/projects/{project_id}/automations/{automation_id}
```

All fields are optional. Only provided fields are updated.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Judge name |
| `template` | string | Evaluation criteria |
| `sample_rate` | float | Span sampling rate |
| `temperature` | float | LLM temperature |
| `evaluation_type` | string | `"binary"` or `"scored"` |
| `score_min` | float | Minimum score |
| `score_max` | float | Maximum score |
| `pass_threshold` | float | Pass/fail threshold |
| `filter` | object | Filter including tag filters |
| `target_prompt_id` | string | Prompt to scope to |

## Link Judge to Prompt

```
POST /signals/projects/{project_id}/prompts/{prompt_id}/judges/{automation_id}/link
```

No request body needed. Links the judge so it only evaluates spans from the target prompt and writes feedback for optimization.

## List Judges

```
GET /signals/projects/{project_id}/automations
```

Returns all judges for the project with evaluation counts and stats.

## Judge Criteria Introspection

```
GET /projects/{project_id}/judges/{judge_id}/criteria
```

Returns the resolved criteria for a scored judge:

```json
{
  "evaluation_type": "scored",
  "criteria": [
    {
      "key": "accuracy",
      "label": "Accuracy",
      "description": "Does the response contain correct information?"
    }
  ]
}
```

ZeroEval infers criteria from the template text when explicit criteria are not provided. For best results, use clearly named criteria with bold labels, section headers, or numbered lists.

## Judge Evaluation Output Schema

### Binary

```json
{
  "reason": "The response only uses information from the provided context.",
  "confidence": 0.92,
  "meets_criteria": true
}
```

### Scored (Single)

```json
{
  "reason": "Thorough answer with relevant examples. Minor gap in edge case coverage.",
  "confidence": 0.85,
  "score": 8.0
}
```

### Scored (Per-Criterion)

```json
{
  "criteria_scores": {
    "accuracy": { "score": 9.0, "reason": "All facts are correct." },
    "empathy": { "score": 7.0, "reason": "Acknowledges frustration but could be warmer." },
    "actionability": { "score": 8.0, "reason": "Clear next steps provided." }
  },
  "score": 8.0,
  "reason": "Strong response with good accuracy and clear actions. Empathy could improve.",
  "confidence": 0.88
}
```

## Filter Validation Rules

- `filter.object_type` must be `"span"` (required)
- `filter.properties.kind` must be `"llm"` (required)
- `filter.properties.sample_rate` is not allowed (use top-level `sample_rate`)
- `filter.properties.tags` (optional): dict of string keys to lists of string values
- `score_min` must be less than `score_max` for scored judges
- `pass_threshold` must be between `score_min` and `score_max` if provided
