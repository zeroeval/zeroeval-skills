# Execution Configuration Reference

Controls for workers, retries, timeouts, checkpoints, repetitions, and resume.

Official docs:
- https://docs.llm-stats.com/python-sdk/evals/repetitions-and-resume
- https://docs.llm-stats.com/python-sdk/evals/common-failures

## ExecutionConfig

Controls how tasks are executed across rows.

```python
from zeroeval import ExecutionConfig

config = ExecutionConfig(
    workers=8,             # parallel workers (default: CPU count)
    max_in_flight=32,      # max concurrent tasks (default: workers * 4)
    timeout_s=90.0,        # per-row timeout in seconds
    retry=RetryPolicy(...),
    failure=FailurePolicy(...),
)

run = dataset.eval(solve, execution=config)
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `workers` | int | CPU count | Number of parallel workers |
| `max_in_flight` | int | workers * 4 | Max tasks queued/running |
| `timeout_s` | float | 90.0 | Per-row timeout (seconds) |
| `retry` | RetryPolicy | see below | Retry configuration |
| `failure` | FailurePolicy | see below | Failure thresholds |

### RetryPolicy

```python
from zeroeval.core.execution import RetryPolicy

policy = RetryPolicy(
    max_attempts=3,         # total attempts per row
    base_delay_s=1.0,       # initial delay between retries
    max_delay_s=30.0,       # max delay cap
    exponential=True,       # exponential backoff
    jitter=True,            # add randomness to avoid thundering herd
    retry_on=(              # exception names to retry on
        "TimeoutError",
        "ConnectionError",
        "RateLimitError",
    ),
)
```

### FailurePolicy

```python
from zeroeval.core.execution import FailurePolicy

policy = FailurePolicy(
    on_row_error="continue",  # "continue" or "stop"
    min_success_rate=0.90,    # run is unhealthy below this
    max_error_rate=0.10,      # run is unhealthy above this
)
```

## CheckpointConfig

Controls how often results are flushed to the backend during execution.

```python
from zeroeval import CheckpointConfig

config = CheckpointConfig(
    enabled=True,              # enable checkpointing
    flush_every_rows=50,       # flush after N completed rows
    flush_every_seconds=10.0,  # flush after N seconds
    persist_errors=True,       # include errored rows in flushes
)

run = dataset.eval(solve, checkpoint=config)
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | True | Enable incremental persistence |
| `flush_every_rows` | int | 50 | Flush cadence (rows) |
| `flush_every_seconds` | float | 10.0 | Flush cadence (seconds) |
| `persist_errors` | bool | True | Include errored rows |

## ResumeConfig

Controls how interrupted or partially-failed runs are resumed.

```python
from zeroeval import ResumeConfig

config = ResumeConfig(
    mode="auto",              # "auto", "force", or "fresh"
    eval_id="eval-abc-123",   # optional: specific eval to resume
    skip_completed=True,      # skip rows that already have results
    retry_errors=False,       # re-run rows that errored
)

run = dataset.eval(solve, resume=config)
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | str | "auto" | `auto`: resume if eval exists. `force`: always resume. `fresh`: always start new. |
| `eval_id` | str | None | Specific eval identifier to resume |
| `skip_completed` | bool | True | Skip rows with existing results |
| `retry_errors` | bool | False | Re-run rows that previously errored (successful rows are still skipped) |

Resume requires:
- Stable `row_id` in every dataset row
- Dataset pushed to the backend (for eval ID resolution)

## Repetitions

Run the same eval multiple times:

```python
run = dataset.eval(solve)
run.repeat(5)  # 5 total runs
```

`repeat(n)` means n total runs (not n additional). Run-level metrics (`@ze.run_metric`) aggregate across repetitions.

## Parameters

Attach metadata to the run for tracking:

```python
run = dataset.eval(solve, parameters={
    "model": "gpt-4o",
    "temperature": 0.7,
    "dataset_version": dataset.version_number,
    "experiment": "prompt-v3",
})
```

Parameters are visible in the eval detail view and useful for filtering/comparing runs.

## Workers Shortcut

Instead of creating an `ExecutionConfig`, pass `workers` directly:

```python
run = dataset.eval(solve, workers=16)
```

This sets `ExecutionConfig.workers=16` and `max_in_flight=64`.

## Common Configuration Patterns

### Fast local testing

```python
run = dataset[:10].eval(solve, workers=1)
```

Slice the dataset to 10 rows, single worker for debugging.

### High-throughput API eval

```python
run = dataset.eval(solve,
    workers=32,
    execution=ExecutionConfig(
        timeout_s=120,
        retry=RetryPolicy(max_attempts=5),
        failure=FailurePolicy(on_row_error="continue"),
    ),
    checkpoint=CheckpointConfig(flush_every_rows=200),
)
```

### Resumable long-running eval

```python
run = dataset.eval(solve,
    resume=ResumeConfig(mode="auto", skip_completed=True),
    checkpoint=CheckpointConfig(flush_every_rows=25),
)
```

If interrupted, re-run the same script — it picks up where it left off.

### Retry only errored rows

```python
# After a run completes with errors (e.g. rate limits, timeouts):
run.resume(retry_errors=True)
```

This re-executes only the rows that errored while skipping successful ones.
The new results overwrite the old errors in the same eval.

Equivalent explicit form:

```python
run = dataset.eval(solve,
    resume=ResumeConfig(mode="force", retry_errors=True, eval_id="<eval-id>"),
)
```

## Debugging with the CLI

Use the CLI to diagnose execution issues after a run completes or fails:

```bash
# Check overall eval health
zeroeval evals summary <eval_id>

# Find errored rows
zeroeval evals results <eval_id> --where "has_error==true"

# Inspect traces for a specific eval
zeroeval evals traces <eval_id> --spans-limit 200

# Get full details in JSON for scripting
zeroeval --output json evals results <eval_id> --where "has_error==true"
```

Common diagnostics:
- **High error rate**: check `FailurePolicy.on_row_error` — set to `"stop"` to halt early, or `"continue"` to collect all errors.
- **Timeouts**: increase `ExecutionConfig.timeout_s` for slow model calls.
- **Rate limits**: increase `RetryPolicy.max_attempts` and `max_delay_s`.
- **Partial runs**: use `ResumeConfig(mode="auto", skip_completed=True)` to continue from where it stopped.
- **Retry errored rows**: use `run.resume(retry_errors=True)` or `ResumeConfig(retry_errors=True)` to re-run only the failed rows from a completed eval.

Full CLI reference: https://docs.llm-stats.com/python-sdk/cli/using-the-cli
