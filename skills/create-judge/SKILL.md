---
name: create-judge
description: This skill should be used when users want to create, design, or configure an automated judge in ZeroEval. It guides through understanding the evaluation goal, choosing binary vs scored evaluation, writing the judge template, designing structured criteria, and creating the judge via dashboard or API. Triggers on "create a judge", "add a judge", "evaluate my LLM output", "set up automated evaluation", "judge template", or "scoring criteria".
---

# Create Judge

Guide users through designing and creating an automated judge that evaluates LLM outputs in ZeroEval.

## When To Use

- Creating a new judge from scratch for any evaluation goal.
- Deciding between binary (pass/fail) and scored (numeric rubric) evaluation.
- Writing judge templates (the evaluation prompt the judge model runs).
- Designing structured criteria for multi-dimensional scored judges.
- Linking a judge to a specific prompt for automatic feedback.
- Troubleshooting judges that aren't producing expected evaluations.

## Execution Sequence

Follow these steps in order. Load reference files only when needed for the current step.

### Step 1: Understand the Evaluation Goal

Ask the user what they want to evaluate and what "good" looks like. Key questions:

- What kind of LLM output are you evaluating? (chat response, extracted data, generated code, etc.)
- What does a good output look like? What makes an output bad?
- Is this a hard pass/fail check or a quality spectrum?
- Should the judge run on all LLM spans or only spans from a specific prompt?

### Step 2: Choose Evaluation Type

Based on the user's answers:

- **Binary** -- use when the criterion is unambiguous (safety, format compliance, factual correctness). The judge returns pass or fail.
- **Scored** -- use when quality is a spectrum (helpfulness, tone, completeness). The judge returns a numeric score.
- **Scored with criteria** -- use when multiple quality dimensions matter. The judge scores each criterion independently plus an overall score.

Read `references/evaluation-patterns.md` for concrete template examples by use case.

### Step 3: Write the Judge Template

The template is the evaluation prompt the judge model uses. Read `references/judge-creation-guide.md` for template conventions and writing guidance.

Key rules:
- Keep templates focused on one evaluation goal.
- Describe quality criteria, not output format.
- For scored judges, define what each score level means.
- For multi-criteria judges, list each criterion with a clear description.

### Step 4: Configure the Judge

Read `references/defaults-and-api-reference.md` for all configurable fields.

Decide:
- **Score range** (scored only): default 0-10 but can be customized.
- **Pass threshold** (scored only): defaults to 70% of scale.
- **Sample rate**: what percentage of spans to evaluate (1.0 = all).
- **Temperature**: default 0.0 (deterministic). Keep at 0.0 for evaluation consistency.
- **Target prompt**: link to a specific prompt to scope evaluations and enable auto-feedback.

### Step 5: Create the Judge

Two options:

**Dashboard**: Navigate to the project, open Judges, click "Create Judge", fill in the fields.

**API**: `POST /signals/projects/{project_id}/automations` with the payload from the defaults reference.

### Step 6: Link to a Prompt (Optional)

If the judge should only evaluate a specific prompt's outputs and feed back into prompt optimization:

```
POST /signals/projects/{project_id}/prompts/{prompt_id}/judges/{automation_id}/link
```

### Step 7: Validate

After creating the judge, confirm it's working:

- Check the Judges section in the dashboard for new evaluations.
- Verify the evaluation count is increasing as new spans arrive.
- Review a few evaluations to confirm the judge is scoring as expected.
- If the judge is linked to a prompt, verify feedback appears in the prompt's feedback tab.

## Key Principles

- **One goal per judge**: a judge that tries to evaluate safety AND helpfulness will do neither well. Create separate judges.
- **Criteria over format**: the template should describe what "good" means, not what the output should look like.
- **Start binary, graduate to scored**: binary judges are easier to validate. Add scored judges once you understand the quality dimensions.
- **Link to prompts**: prompt-linked judges enable the optimization loop. Always link when using `ze.prompt`.
- **Low temperature**: keep at 0.0 for reproducible evaluations.
