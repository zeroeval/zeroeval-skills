# Task and Evaluation Patterns

Common patterns for writing tasks and evaluators with the ZeroEval SDK.

Official docs:
- https://docs.llm-stats.com/python-sdk/evals/running-evals
- https://docs.llm-stats.com/python-sdk/evals/scoring-and-metrics
- https://docs.llm-stats.com/python-sdk/evals/signals-and-traces

## Task Contract

A `@ze.task` function must:
1. Accept a single `row` argument (a `DotDict` with dot and dict access).
2. Return a dict with keys matching the declared `outputs`.

```python
@ze.task(outputs=["prediction", "confidence"])
def solve(row):
    result = call_model(row.question)
    return {
        "prediction": result.text,
        "confidence": result.score,
    }
```

Optional: give the task a custom name:

```python
@ze.task(outputs=["answer"], name="gpt4o_solver")
def solve_with_gpt4o(row):
    return {"answer": gpt4o(row.question)}
```

## Row Evaluations

Run once per row. Use for per-example scores.

```python
@ze.evaluation(mode="row", outputs=["exact_match"])
def exact_match(row):
    return {"exact_match": int(row.prediction == row.answer)}
```

### Exact match

```python
@ze.evaluation(mode="row", outputs=["exact_match"])
def exact_match(row):
    return {"exact_match": int(row.prediction.strip().lower() == row.answer.strip().lower())}
```

### Contains keyword

```python
@ze.evaluation(mode="row", outputs=["contains_keyword"])
def contains_keyword(row):
    return {"contains_keyword": int(row.expected_keyword.lower() in row.prediction.lower())}
```

### Regex match

```python
import re

@ze.evaluation(mode="row", outputs=["format_valid"])
def format_valid(row):
    pattern = r"^\d{4}-\d{2}-\d{2}$"
    return {"format_valid": int(bool(re.match(pattern, row.prediction)))}
```

### Numeric comparison

```python
@ze.evaluation(mode="row", outputs=["within_tolerance"])
def within_tolerance(row):
    try:
        pred = float(row.prediction)
        expected = float(row.answer)
        return {"within_tolerance": int(abs(pred - expected) < 0.01)}
    except (ValueError, TypeError):
        return {"within_tolerance": 0}
```

### LLM-as-judge (row level)

```python
@ze.evaluation(mode="row", outputs=["quality_score"])
def llm_judge(row):
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"""Rate this answer on a scale of 1-5.

Question: {row.question}
Expected: {row.answer}
Got: {row.prediction}

Reply with just the number."""
        }],
        temperature=0,
    )
    score = int(response.choices[0].message.content.strip())
    return {"quality_score": score}
```

## Column Evaluations

Run once with all rows. Use for aggregate metrics.

```python
@ze.column_metric(outputs=["accuracy"])
def accuracy(rows):
    correct = sum(1 for r in rows if r["prediction"] == r["answer"])
    return {"accuracy": correct / len(rows)}
```

### F1 score

```python
@ze.column_metric(outputs=["f1"])
def f1_score(rows):
    tp = sum(1 for r in rows if r["prediction"] == "yes" and r["answer"] == "yes")
    fp = sum(1 for r in rows if r["prediction"] == "yes" and r["answer"] == "no")
    fn = sum(1 for r in rows if r["prediction"] == "no" and r["answer"] == "yes")
    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0
    return {"f1": f1}
```

### Mean score

```python
@ze.column_metric(outputs=["avg_quality"])
def avg_quality(rows):
    scores = [r["quality_score"] for r in rows if r.get("quality_score") is not None]
    return {"avg_quality": sum(scores) / len(scores) if scores else 0}
```

## Run Evaluations

Run once across all repetitions. Use for metrics that compare repeated runs.

```python
@ze.run_metric(outputs=["pass_at_3"])
def pass_at_3(all_runs):
    passed = 0
    for row_group in all_runs:
        if any(r.get("exact_match") == 1 for r in row_group):
            passed += 1
    return {"pass_at_3": passed / len(all_runs) if all_runs else 0}
```

## column_map

`column_map` binds evaluator parameter names to actual column names. This decouples evaluator logic from column naming.

```python
# The evaluator uses generic names
@ze.evaluation(mode="row", outputs=["match"])
def check_match(row):
    return {"match": int(row.predicted == row.expected)}

# column_map maps them to the actual columns
run.score([check_match], column_map={
    "predicted": "model_output",    # evaluator's "predicted" reads from "model_output"
    "expected": "ground_truth",     # evaluator's "expected" reads from "ground_truth"
})
```

Without `column_map`, the evaluator reads columns by the exact names used in the function. With it, you can reuse the same evaluator across benchmarks with different column naming.

## Signal Emission Patterns

Signals capture runtime facts during task execution. They don't affect scoring — they're metadata for debugging and analysis.

### RAG pipeline

```python
@ze.task(outputs=["answer"])
def rag_answer(row):
    docs = retrieve(row.question)
    ze.emit_signal("retrieved_doc_count", len(docs))
    ze.emit_signal("top_doc_relevance", docs[0].score if docs else 0.0)
    ze.emit_signal("retrieval_source", docs[0].source if docs else "none")

    answer = generate(row.question, docs)
    ze.emit_signal("answer_length", len(answer))
    return {"answer": answer}
```

### Code generation

```python
@ze.task(outputs=["code"])
def generate_code(row):
    code = call_model(row.prompt)
    ze.emit_signal("has_syntax_error", not compiles(code))
    ze.emit_signal("line_count", code.count("\n") + 1)
    ze.emit_signal("uses_imports", "import" in code)
    return {"code": code}
```

### Multi-step agent

```python
@ze.task(outputs=["result"])
def agent_task(row):
    steps = 0
    for action in agent.run(row.instruction):
        steps += 1
        ze.emit_signal("step_type", action.type)

    ze.emit_signal("total_steps", steps)
    ze.emit_signal("completed", agent.success)
    return {"result": agent.output}
```

### Signal types

| Python type | Signal type |
|-------------|-------------|
| `bool` | `boolean` |
| `int`, `float` | `numerical` |
| `str` | `categorical` |

Types are auto-inferred from the value.

## Applying Multiple Evaluators

Chain multiple evaluators in a single `score()` call:

```python
run.score(
    [exact_match, format_valid, llm_judge, accuracy],
    column_map={
        "prediction": "prediction",
        "answer": "answer",
    },
)
```

Row-level evaluators run per row, column-level run once on all rows, run-level run once across repetitions. Mix them freely.
