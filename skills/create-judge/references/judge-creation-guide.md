# Judge Creation Guide

How to write effective judge templates and create judges in ZeroEval.

## How Judges Work

When a traced LLM span arrives, ZeroEval:

1. Matches the span against the judge's filter.
2. Combines the span's input and output with the judge template.
3. Sends the assembled evaluation to a dedicated evaluation model.
4. Returns a pass/fail decision (binary) or a numeric score (scored).
5. If the judge is linked to a prompt, writes feedback for optimization.

You write the **template** (the evaluation criteria). ZeroEval handles assembly, model selection, and result storage.

## Filter Contract

Every judge must use this filter:

```json
{
  "object_type": "span",
  "properties": {
    "kind": "llm"
  }
}
```

Optional: add tag filters to narrow scope:

```json
{
  "object_type": "span",
  "properties": {
    "kind": "llm",
    "tags": {
      "feature": ["customer-support"]
    }
  }
}
```

Do not put `sample_rate` inside `filter.properties` -- use the top-level `sample_rate` field.

## Writing Templates

The template is the evaluation criteria you write. ZeroEval automatically combines it with the span's input and output before evaluation.

### Binary Templates

Write a clear description of what "pass" means:

```
Evaluate whether the response directly answers the user's question
without introducing information not present in the provided context.

Pass if the response:
- Answers the question asked
- Uses only information from the context
- Does not hallucinate facts

Fail if the response:
- Ignores the question
- Introduces unsupported claims
- Contradicts the provided context
```

### Scored Templates (Single Score)

Write a rubric describing what each quality level means:

```
Rate the helpfulness of the response on a scale from 0 to 10.

0-2: Unhelpful. Does not address the question or provides incorrect information.
3-4: Partially helpful. Addresses part of the question but misses key points.
5-6: Adequate. Answers the question but lacks depth or specificity.
7-8: Helpful. Thorough answer with relevant details and context.
9-10: Excellent. Comprehensive, well-structured, anticipates follow-up questions.
```

### Scored Templates (Per-Criterion)

Same as single score, but add named criteria. ZeroEval can infer per-criterion scoring from clearly structured rubrics. Use bold labels, section headers, or numbered lists so each criterion is easy to identify.

Example rubric:

```
Rate the quality of the customer support response.

**Accuracy**: Does the response contain correct information about our products and policies?
**Empathy**: Does the response acknowledge the customer's frustration and show understanding?
**Actionability**: Does the response provide clear next steps the customer can follow?
**Tone**: Is the response professional, friendly, and appropriate for the context?
```

This defines four criteria (`accuracy`, `empathy`, `actionability`, `tone`) that are scored independently plus an overall score.

### Template Writing Tips

- Describe quality criteria, not output format. The judge evaluates the LLM's response, not its structure.
- Keep templates focused on one evaluation goal per judge.
- For scored judges, describe what each score level means. Vague rubrics produce inconsistent scores.
- Avoid mentioning "output format" or "JSON" in your template -- ZeroEval handles response formatting separately. If the template mentions "Output format," it may be confused with the content being evaluated.
- Test with `debug` mode first to see what the judge produces before enabling at full sample rate.

## Structured Criteria

For per-criterion judges, criteria can also be defined explicitly. Each criterion has:

- `key` (required): machine-readable identifier, lowercase with underscores
- `label` (required): human-readable display name
- `description` (optional): what the criterion measures
- `order` (optional): display ordering

When explicit criteria are provided, they are used instead of inferring criteria from the template text.

## Linking Judges to Prompts

Set `target_prompt_id` during creation or link after creation:

```
POST /signals/projects/{project_id}/prompts/{prompt_id}/judges/{automation_id}/link
```

What linking enables:
- The judge only evaluates spans produced by the target prompt.
- After each evaluation, the judge automatically writes feedback to the prompt.
- This feedback powers prompt optimization in the Prompt Library.

Always link judges when using `ze.prompt` -- it closes the evaluate-optimize loop.

## Creation via API

```
POST /signals/projects/{project_id}/automations
```

Minimal binary judge:

```json
{
  "name": "Hallucination Check",
  "sample_rate": 1.0,
  "filter": {
    "object_type": "span",
    "properties": {
      "kind": "llm"
    }
  },
  "template": "Does the response contain claims not supported by the provided context? Pass if grounded, fail if hallucinated."
}
```

Minimal scored judge:

```json
{
  "name": "Helpfulness Score",
  "sample_rate": 1.0,
  "filter": {
    "object_type": "span",
    "properties": {
      "kind": "llm"
    }
  },
  "template": "Rate how helpful and complete the response is. 0-3: unhelpful, 4-6: adequate, 7-10: excellent.",
  "evaluation_type": "scored"
}
```

ZeroEval auto-fills defaults for `score_min` (0.0), `score_max` (10.0), `pass_threshold` (7.0), `temperature` (0.0), and the evaluation model.

## Creation via Dashboard

1. Open the project in the ZeroEval dashboard.
2. Navigate to Judges in the sidebar.
3. Click "Create Judge."
4. Fill in: name, template, evaluation type (binary or scored).
5. For scored: set score range and pass threshold (or accept defaults).
6. Optionally link to a prompt.
7. Save.

The judge begins evaluating new spans immediately based on the sample rate.
