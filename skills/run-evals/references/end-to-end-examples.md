# End-to-End Evaluation Examples

Complete working examples of evaluation pipelines with the ZeroEval SDK.

Official examples:
- https://docs.llm-stats.com/python-sdk/examples/text-eval-end-to-end
- https://docs.llm-stats.com/python-sdk/examples/multimodal-eval-end-to-end

## Example 1: Text QA Benchmark

Evaluate a model on question-answering with exact match and accuracy scoring.

```python
import zeroeval as ze
import openai

ze.init()

# Pull the benchmark
dataset = ze.Dataset.pull("math-qa", subset="default")

# Define the task
@ze.task(outputs=["prediction"])
def solve(row):
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": row.question}],
        temperature=0,
    )
    return {"prediction": response.choices[0].message.content.strip()}

# Define evaluators
@ze.evaluation(mode="row", outputs=["exact_match"])
def exact_match(row):
    pred = row.prediction.strip().lower()
    expected = row.answer.strip().lower()
    return {"exact_match": int(pred == expected)}

@ze.column_metric(outputs=["accuracy"])
def accuracy(rows):
    correct = sum(1 for r in rows if r["exact_match"] == 1)
    return {"accuracy": correct / len(rows)}

# Run and score
run = dataset.eval(solve, workers=8, parameters={"model": "gpt-4o"})
run.score([exact_match, accuracy], column_map={
    "prediction": "prediction",
    "answer": "answer",
})

print(f"Eval ID: {run.id}")
print(f"Accuracy: {run.metrics}")
```

## Example 2: LLM-as-Judge Evaluation

Use a stronger model to score the quality of responses.

```python
import zeroeval as ze
import openai

ze.init()

dataset = ze.Dataset.pull("customer-support-qa")

@ze.task(outputs=["response"])
def support_agent(row):
    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a helpful customer support agent."},
            {"role": "user", "content": row.customer_query},
        ],
    )
    return {"response": response.choices[0].message.content}

@ze.evaluation(mode="row", outputs=["helpfulness", "accuracy_score"])
def judge_quality(row):
    judgment = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"""Score this customer support response.

Customer query: {row.customer_query}
Expected answer: {row.expected_answer}
Agent response: {row.response}

Rate helpfulness (1-5) and factual accuracy (1-5).
Reply in format: helpfulness=N accuracy=N"""
        }],
        temperature=0,
    )
    text = judgment.choices[0].message.content
    helpfulness = int(text.split("helpfulness=")[1][0])
    accuracy_score = int(text.split("accuracy=")[1][0])
    return {"helpfulness": helpfulness, "accuracy_score": accuracy_score}

@ze.column_metric(outputs=["avg_helpfulness", "avg_accuracy"])
def aggregate_scores(rows):
    h = [r["helpfulness"] for r in rows if r.get("helpfulness")]
    a = [r["accuracy_score"] for r in rows if r.get("accuracy_score")]
    return {
        "avg_helpfulness": sum(h) / len(h) if h else 0,
        "avg_accuracy": sum(a) / len(a) if a else 0,
    }

run = dataset.eval(support_agent, workers=4, parameters={"model": "gpt-4o-mini"})
run.score([judge_quality, aggregate_scores], column_map={
    "response": "response",
    "customer_query": "customer_query",
    "expected_answer": "expected_answer",
})
```

## Example 3: Multi-Model Comparison

Compare multiple models on the same benchmark.

```python
import zeroeval as ze
import openai

ze.init()

dataset = ze.Dataset.pull("coding-problems", version_number=2)

def make_task(model_name):
    @ze.task(outputs=["code"], name=f"{model_name}_solver")
    def solve(row):
        response = openai.chat.completions.create(
            model=model_name,
            messages=[{"role": "user", "content": row.prompt}],
            temperature=0,
        )
        return {"code": response.choices[0].message.content}
    return solve

@ze.evaluation(mode="row", outputs=["passes"])
def test_code(row):
    try:
        exec(row.code)
        result = eval(row.test_expression)
        return {"passes": int(result == row.expected)}
    except Exception:
        return {"passes": 0}

@ze.column_metric(outputs=["pass_rate"])
def pass_rate(rows):
    passed = sum(1 for r in rows if r["passes"] == 1)
    return {"pass_rate": passed / len(rows)}

models = ["gpt-4o", "gpt-4o-mini", "claude-3-5-sonnet-20241022"]

for model in models:
    task = make_task(model)
    run = dataset.eval(task, workers=8, parameters={"model": model})
    run.score([test_code, pass_rate])
    print(f"{model}: {run.metrics}")
```

## Example 4: RAG Pipeline with Signals

Evaluate a RAG pipeline while emitting signals for debugging.

```python
import zeroeval as ze

ze.init()

dataset = ze.Dataset.pull("rag-qa")

@ze.task(outputs=["answer"])
def rag_pipeline(row):
    # Retrieve
    docs = vector_search(row.question, top_k=5)
    ze.emit_signal("retrieved_count", len(docs))
    ze.emit_signal("top_similarity", docs[0].score if docs else 0.0)

    # Filter
    relevant = [d for d in docs if d.score > 0.7]
    ze.emit_signal("relevant_count", len(relevant))

    # Generate
    context = "\n".join(d.text for d in relevant)
    answer = generate_answer(row.question, context)
    ze.emit_signal("answer_length", len(answer))
    ze.emit_signal("used_context", len(relevant) > 0)

    return {"answer": answer}

@ze.evaluation(mode="row", outputs=["faithfulness"])
def check_faithfulness(row):
    return {"faithfulness": int(is_grounded(row.answer, row.reference_answer))}

@ze.column_metric(outputs=["avg_faithfulness"])
def avg_faithfulness(rows):
    scores = [r["faithfulness"] for r in rows]
    return {"avg_faithfulness": sum(scores) / len(scores) if scores else 0}

run = dataset.eval(rag_pipeline, workers=4)
run.score([check_faithfulness, avg_faithfulness])
```

## Example 5: Resumable Long-Running Eval

```python
import zeroeval as ze

ze.init()

dataset = ze.Dataset.pull("large-benchmark")

@ze.task(outputs=["result"])
def expensive_task(row):
    return {"result": run_expensive_model(row.input)}

# First run (may be interrupted)
run = dataset.eval(
    expensive_task,
    workers=4,
    execution=ze.ExecutionConfig(timeout_s=300),
    checkpoint=ze.CheckpointConfig(flush_every_rows=10),
    resume=ze.ResumeConfig(mode="auto", skip_completed=True),
)

# If interrupted, re-run the exact same script.
# It automatically skips completed rows and continues.

# If the run completed but some rows errored (e.g. rate limits),
# retry only the failed rows:
run.resume(retry_errors=True)
```
