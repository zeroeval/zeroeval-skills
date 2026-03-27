# Git-Based Data Workflow

Complete guide for managing benchmark data via git. Use this path for smaller benchmark repos and metadata-heavy workflows.

Official docs: https://docs.llm-stats.com/python-sdk/datasets/uploading-data#git-recommended

## When To Use Git

- Full version history with meaningful commits
- Include non-data files (scorers, README, configs) alongside data
- Standard tooling — works with any git client
- Each push creates a new dataset version automatically

For large parquet uploads, prefer the upload session flow instead of a single large Git push.

## Clone the Repository

Every benchmark has a git repository:

```bash
git clone https://git.llm-stats.com/<org>/<benchmark-slug>.git
cd <benchmark-slug>
```

### Authentication

Use your ZeroEval API key as the git password. Username can be anything:

```bash
git clone https://user:<your-api-key>@git.llm-stats.com/<org>/<benchmark-slug>.git
```

Find the clone URL on the benchmark's Files tab in the hub.

## Repository Layout

```
my-benchmark/
├── data/
│   ├── default.parquet     # main subset
│   ├── easy.parquet        # optional: named subset
│   └── hard.parquet        # optional: named subset
├── scorers/
│   └── exact_match.py      # auto-detected scorer implementation
├── README.md               # displayed on benchmark overview page
└── requirements.txt        # optional dependencies
```

### Rules

- **Only `.parquet` files in `data/` are detected as subsets.** CSV and JSONL files are stored but not auto-indexed.
- Each Parquet filename (without extension) becomes the subset name.
- If you have one subset, name it `data/default.parquet`.
- `scorers/*.py` files are auto-detected as scorer implementations.
- `README.md` is shown on the benchmark overview page.
- Any other files are stored and browsable in the Files tab.

## Converting Data to Parquet

### From CSV with pandas

```python
import pandas as pd

df = pd.read_csv("questions.csv")
df.to_parquet("data/default.parquet", index=False)
```

### From CSV with PyArrow

```python
import pyarrow.csv as pcsv
import pyarrow.parquet as pq

table = pcsv.read_csv("questions.csv")
pq.write_table(table, "data/default.parquet")
```

### From JSONL with pandas

```python
import pandas as pd

df = pd.read_json("questions.jsonl", lines=True)
df.to_parquet("data/default.parquet", index=False)
```

### Creating multiple subsets

```python
import pandas as pd

df = pd.read_csv("questions.csv")

easy = df[df["difficulty"] == "easy"]
hard = df[df["difficulty"] == "hard"]

easy.to_parquet("data/easy.parquet", index=False)
hard.to_parquet("data/hard.parquet", index=False)
```

## Commit and Push

```bash
git add .
git commit -m "Add benchmark data"
git push
```

`git push` updates the benchmark repo immediately, then the platform indexes the dataset asynchronously. Check the benchmark page or run `zeroeval datasets ingest <slug>` if you need to confirm whether the latest push is still `received`, currently `indexing`, `ready`, or `failed`.

Each successful ingest triggers the platform to:
- Index all files
- Detect subsets from `data/*.parquet`
- Detect scorers from `scorers/*.py`
- Update schema and column types from Parquet Arrow metadata
- Create a new dataset version

## Subset Detection

| File path | Subset name |
|-----------|-------------|
| `data/default.parquet` | `default` |
| `data/easy.parquet` | `easy` |
| `data/hard.parquet` | `hard` |
| `data/gpqa_diamond.parquet` | `gpqa_diamond` |

Default subset resolution order:
1. Benchmark's configured default subset
2. `default.parquet`
3. Single subset (if only one exists)
4. First alphabetically

## Pulling Subsets in Code

After pushing via git, pull subsets in the SDK:

```python
import zeroeval as ze

ze.init()

easy = ze.Dataset.pull("my-benchmark", subset="easy")
hard = ze.Dataset.pull("my-benchmark", subset="hard")
```

## Best Practices

- **Always include a `row_id` column** in your Parquet files for stable row identity.
- **Use meaningful subset names** that describe the data split (e.g., `train`, `test`, `easy`, `hard`).
- **Commit data and scorers together** so each version is self-contained.
- **Write a README.md** explaining the benchmark, data format, and expected columns.
- **Pin versions in eval scripts**: `ze.Dataset.pull("name", version_number=N)`.

## Private Benchmarks

For private benchmarks, authentication is required for both clone and push:

```bash
git clone https://user:<api-key>@git.llm-stats.com/<org>/<slug>.git
```

Public benchmarks can be cloned without authentication.
