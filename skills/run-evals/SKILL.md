---
name: run-evals
description: Write tasks, evaluations, and scoring pipelines with the ZeroEval Python SDK. Covers defining @ze.task functions, running evals with dataset.eval(), writing row/column/run evaluators, scoring with column_map, emitting signals, configuring execution (workers, retries, checkpoints), repeating and resuming runs, and inspecting results. Triggers on "run evals", "write evaluation", "benchmark model", "score results", "evaluation pipeline", "task decorator", "scoring function", "column_map", "emit signal", "resume eval", "repeat eval".
---

# Run Evals

Guide users through writing tasks, running evaluations, scoring results, and configuring execution with the ZeroEval Python SDK.

## When To Use

- Defining a `@ze.task` to run against a benchmark dataset.
- Running evals with `dataset.eval()`.
- Writing row-level, column-level, or run-level evaluations.
- Using `column_map` to bind evaluator arguments to dataset columns.
- Emitting runtime signals during task execution.
- Configuring workers, retries, timeouts, and checkpoints.
- Repeating evals or resuming interrupted runs.
- Inspecting eval results, metrics, and health.

## Execution Sequence

Follow these steps in order. Load reference files for detailed patterns and configuration.

### Step 1: Install and Initialize

Check if the SDK is installed. If not, install it:

```bash
pip install zeroeval
```

Verify the CLI is available:

```bash
zeroeval --help
```

Set the API key (if not already configured):

```bash
export ZEROEVAL_API_KEY="sk_ze_..."
# or use the interactive setup:
zeroeval setup
```

Initialize in code and pull the dataset:

```python
import zeroeval as ze

ze.init()

dataset = ze.Dataset.pull("my-benchmark")
```

If the dataset doesn't exist yet, use the `manage-data` skill to create and push it first.

Full setup guide: https://docs.llm-stats.com/python-sdk/installation

### Step 2: Define a Task

A task is a function decorated with `@ze.task` that processes one row and returns a dict of outputs.

```python
@ze.task(outputs=["prediction"])
def solve(row):
    response = call_llm(row.question)
    return {"prediction": response}
```

Rules:
- `outputs` declares the column names this task produces.
- The function receives a single `row` (DotDict) with all dataset columns.
- Must return a dict with keys matching `outputs`.
- Can optionally emit signals during execution (see Step 5).

### Step 3: Run the Eval

```python
run = dataset.eval(solve)
```

This executes `solve` on every row, streams results to the backend, and returns an `Eval` object.

Options:

```python
run = dataset.eval(
    solve,
    workers=8,                          # parallel workers
    parameters={"model": "gpt-4o"},     # metadata attached to the run
    execution=ze.ExecutionConfig(...),   # advanced execution controls
    checkpoint=ze.CheckpointConfig(...), # persistence cadence
)
```

Full reference: https://docs.llm-stats.com/python-sdk/evals/running-evals

### Step 4: Score Results

Define evaluators and apply them to the run.

```python
# Row-level: runs per row
@ze.evaluation(mode="row", outputs=["exact_match"])
def exact_match(row):
    return {"exact_match": int(row.prediction == row.answer)}

# Column-level: aggregate over all rows
@ze.column_metric(outputs=["accuracy"])
def accuracy(rows):
    correct = sum(1 for r in rows if r["prediction"] == r["answer"])
    return {"accuracy": correct / len(rows)}

# Apply evaluators
run.score([exact_match, accuracy], column_map={
    "prediction": "prediction",
    "answer": "answer",
})
```

`column_map` binds evaluator parameter names to actual column names in the dataset. This decouples evaluator logic from column naming.

`run.score()` is an alias for `run.eval()`.

For detailed patterns (row, column, run evaluators, LLM-as-judge, regex), read `references/task-and-eval-patterns.md`.

Full reference: https://docs.llm-stats.com/python-sdk/evals/scoring-and-metrics

### Step 5: Emit Signals (Optional)

Signals capture runtime facts during task execution — distinct from task outputs and evaluation scores.

```python
@ze.task(outputs=["answer"])
def rag_answer(row):
    docs = retrieve(row.question)
    ze.emit_signal("retrieved_doc_count", len(docs))
    ze.emit_signal("top_doc_score", docs[0].score if docs else 0.0)

    answer = generate(row.question, docs)
    ze.emit_signal("answer_length", len(answer))
    return {"answer": answer}
```

Signal types: `boolean`, `numerical`, `categorical` (auto-inferred from value type).

Signals attach to the current span and persist with traces. They surface in the eval detail UI.

Full reference: https://docs.llm-stats.com/python-sdk/evals/signals-and-traces

### Step 6: Configure Execution (Optional)

For advanced control, read `references/execution-config.md`.

Quick overview:

```python
run = dataset.eval(
    solve,
    execution=ze.ExecutionConfig(
        workers=16,
        timeout_s=120,
    ),
    checkpoint=ze.CheckpointConfig(
        flush_every_rows=100,
    ),
)
```

Full reference: https://docs.llm-stats.com/python-sdk/evals/repetitions-and-resume

### Step 7: Repeat, Resume, and Retry Errors (Optional)

```python
# Run the eval 5 times total
run.repeat(5)

# Resume an interrupted run (skips already-completed rows)
run.resume()

# Retry only errored rows (e.g. after rate limits or timeouts)
run.resume(retry_errors=True)
```

Resume requires stable `row_id` in the dataset. Read `references/execution-config.md` for `ResumeConfig` options.

### Step 8: Inspect Results

In code:

```python
print(run.id)          # backend eval ID
print(run.rows)        # list of result rows
print(run.metrics)     # computed metrics
print(run.health)      # success rate, error count, timing
```

Results sync to the benchmark hub automatically — visible in the Evals tab.

### Step 9: Inspect with the CLI

The CLI is the fastest way to check eval status, debug errors, and pull results without writing scripts:

```bash
# List recent evals
zeroeval evals list --status completed --limit 10

# Get eval summary (status, row counts, timing)
zeroeval evals summary <eval_id>

# View results for a specific eval
zeroeval evals results <eval_id> --limit 20

# Check scorers applied to this eval
zeroeval evals scorers <eval_id>

# Pull scores for a specific scorer
zeroeval evals scores <eval_id> <scorer_id>

# Inspect traces (spans, signals)
zeroeval evals traces <eval_id> --spans-limit 100

# Filter results — show only errors
zeroeval evals results <eval_id> --where "has_error==true"

# Select specific fields
zeroeval evals results <eval_id> \
  --select "row_id,result.prediction,result.exact_match" \
  --order "row_id:asc"

# JSON output for scripting
zeroeval --output json evals summary <eval_id>
```

Always run `zeroeval evals --help` or `zeroeval evals results --help` to discover available flags in your installed version.

Full CLI reference: https://docs.llm-stats.com/python-sdk/cli/using-the-cli

### Step 10: Push Eval Scripts to the Repo (Optional)

Once you're happy with your task and evaluators, push the scripts to the benchmark's git repo so they're versioned alongside the data.

```bash
# Clone the repo (if not already cloned)
git clone "https://user:YOUR_API_KEY@<git-url>"
cd <benchmark-slug>

# Add your eval scripts
mkdir -p scorers
cp my_task.py scorers/
cp my_evaluators.py scorers/

git add . && git commit -m "Add eval scripts" && git push
```

Files in `scorers/*.py` are auto-detected by the platform and shown in the benchmark config. You can also add any supporting files (configs, requirements, notebooks) — they're stored and browsable in the Files tab.

This step is optional and only recommended once the eval pipeline is stable. For the git clone URL and auth details, refer to the `manage-data` skill or the benchmark's Files tab.

### Step 11: Validate

- [ ] Task returns a dict with all declared `outputs` keys
- [ ] Eval completes without unhandled errors (`zeroeval evals summary <id>`)
- [ ] Results visible in the benchmark hub (Evals tab)
- [ ] Metrics appear in the eval detail view
- [ ] No unexpected errors (`zeroeval evals results <id> --where "has_error==true"`)
- [ ] Signals appear in the eval detail (if emitted)

## Key Principles

- **Outputs are the primary data.** Task outputs are the columns produced by your task function. They become the scored data.
- **Signals are runtime facts.** Use `ze.emit_signal()` for metadata captured during execution (latency, doc count, phase). They don't affect scoring.
- **Evaluations are post-hoc scoring.** Row, column, and run evaluators score the results after the task has run. They produce metrics.
- **`column_map` decouples naming.** Evaluators use generic parameter names; `column_map` maps them to actual columns. This makes evaluators reusable across benchmarks.
- **Start simple, add complexity.** Begin with a task + one row evaluator. Add column metrics, signals, and config as needed.
- **Pin dataset versions.** Use `version_number` when pulling datasets in eval scripts for reproducibility.
