# Judges Playbook

Quick guide for choosing which judges to set up first. For detailed judge creation (templates, API, criteria design), use the **create-judge** skill.

## What Are Judges

Judges are automated evaluators that score LLM outputs on every traced span. They run on the backend against spans with `kind: "llm"` and produce binary (pass/fail) or scored (numeric rubric) evaluations.

## Judge Types

### Binary Judges

Return `pass` or `fail` for each span. Best for hard compliance checks.

- Use when: the quality criterion is unambiguous (factual correctness, safety, format compliance)

### Scored Judges

Return a numeric score within a configurable range. Best for nuanced quality rubrics.

- Default score range: 0 to 10, pass threshold at 7 (70% of scale)
- Use when: quality is a spectrum (helpfulness, tone, completeness)
- Support per-criterion breakdowns for multi-dimensional evaluation

---

## Starter Judge Recommendations by App Pattern

### Customer Support / Chat Agents

| Judge Name | Type | Template Guidance |
|-----------|------|-------------------|
| Response Helpfulness | scored | "Rate how helpful and complete the response is for the customer's question." |
| Tone Compliance | binary | "Does the response maintain a professional and empathetic tone?" |
| Hallucination Check | binary | "Does the response contain claims not supported by the provided context or knowledge base?" |

### Extraction / Classification Pipelines

| Judge Name | Type | Template Guidance |
|-----------|------|-------------------|
| Extraction Accuracy | binary | "Does the output contain all required fields extracted correctly from the input?" |
| Format Compliance | binary | "Does the output match the expected JSON/structured format?" |
| Completeness Score | scored | "Rate how completely the output captures all relevant information from the input." |

### Coding Copilots

| Judge Name | Type | Template Guidance |
|-----------|------|-------------------|
| Code Correctness | binary | "Does the generated code compile/run without errors and produce the expected output?" |
| Code Quality Score | scored | "Rate the code for readability, idiomatic patterns, and appropriate error handling." |
| Security Check | binary | "Does the generated code introduce any security vulnerabilities (injection, hardcoded secrets, unsafe operations)?" |

### Retrieval QA / RAG Assistants

| Judge Name | Type | Template Guidance |
|-----------|------|-------------------|
| Answer Groundedness | binary | "Is the answer fully supported by the retrieved context? No external knowledge used." |
| Relevance Score | scored | "Rate how relevant and directly useful the answer is to the user's question." |
| Citation Accuracy | binary | "Does the answer correctly reference the source documents it claims to cite?" |

---

## Getting Started Checklist

- [ ] Choose 1-2 starter judges from the recommendations above based on your app pattern.
- [ ] Use the **create-judge** skill for step-by-step guidance on writing templates, setting configuration, and creating the judge via dashboard or API.
- [ ] Link judges to relevant prompts via `target_prompt_id` if using `ze.prompt`.
- [ ] Confirm evaluations appear in the dashboard after new spans are ingested.
