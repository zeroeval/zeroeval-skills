# Dataset Lifecycle Reference

Complete API reference for creating, modifying, pushing, pulling, and inspecting datasets with the ZeroEval Python SDK.

Official docs: https://docs.llm-stats.com/python-sdk/datasets/creating

## Creating Datasets

### From Python dicts

```python
import zeroeval as ze

ze.init()

dataset = ze.Dataset("my-benchmark", data=[
    {"row_id": "q1", "question": "What is 2+2?", "answer": "4"},
    {"row_id": "q2", "question": "Capital of France?", "answer": "Paris"},
], description="Math and geography questions")
```

- First argument is the dataset name (used as the slug on the backend).
- `data` must be a list of dicts. All items must be dictionaries.
- `description` is optional but recommended.
- Include a `row_id` field for stable identity across versions.

### From CSV

```python
dataset = ze.Dataset("/path/to/questions.csv")
```

- The filename (without extension) becomes the dataset name.
- CSV is parsed with `csv.DictReader` — first row must be headers.
- All values are strings by default. Convert types after loading if needed.

## Modifying Data

### Add rows

```python
dataset.add_rows([
    {"row_id": "q3", "question": "Largest planet?", "answer": "Jupiter"},
    {"row_id": "q4", "question": "H2O?", "answer": "Water"},
])
```

### Update a row

```python
dataset.update_row(0, {"row_id": "q1", "question": "Updated question?", "answer": "Yes"})
```

Replaces the entire row at the given index.

### Delete a row

```python
dataset.delete_row(2)  # deletes the third row (0-indexed)
```

### Indexing and iteration

```python
row = dataset[0]            # single row (DotDict)
subset = dataset[0:5]       # sliced Dataset
print(len(dataset))         # row count
print(dataset.columns)      # list of column names

for row in dataset:
    print(row.question)     # dot notation
    print(row["answer"])    # dict notation
```

## Pushing Data

```python
dataset.push()
```

- Creates the dataset on the backend if it doesn't exist.
- Auto-creates a new version if the dataset already exists.
- After push: `dataset.backend_id`, `dataset.version_id`, `dataset.version_number` are populated.
- Returns `self` for chaining: `dataset.push().eval(task)`.

## Pulling Data

```python
# Latest version
dataset = ze.Dataset.pull("benchmark-name")

# Pinned version
dataset = ze.Dataset.pull("benchmark-name", version_number=3)

# Specific subset
dataset = ze.Dataset.pull("benchmark-name", subset="hard")
```

Official docs: https://docs.llm-stats.com/python-sdk/datasets/loading

### Default subset resolution

When no subset is specified:
1. Benchmark's configured default subset
2. A subset named `default`
3. If only one subset exists, that one
4. First alphabetically

### Properties after pull

- `dataset.name` — dataset name
- `dataset.data` — list of row dicts
- `dataset.columns` — list of column names
- `dataset.backend_id` — backend dataset ID
- `dataset.version_id` — version ID
- `dataset.version_number` — version number

## Versioning

Official docs: https://docs.llm-stats.com/python-sdk/datasets/versioning-and-subsets

Every `push()` creates a new version. Pin versions in eval scripts for reproducibility:

```python
dataset = ze.Dataset.pull("my-benchmark", version_number=5)
run = dataset.eval(my_task)
```

Track version in run parameters:

```python
run = dataset.eval(my_task, parameters={
    "dataset_version": dataset.version_number,
    "subset": "hard",
})
```

## Multimodal Data

Official docs: https://docs.llm-stats.com/python-sdk/datasets/multimodal

### Adding media from files

```python
dataset.add_image(row_index, "column_name", "/path/to/image.jpg")
dataset.add_audio(row_index, "column_name", "/path/to/audio.wav")
dataset.add_video(row_index, "column_name", "/path/to/video.mp4")
```

Files are encoded as data URIs (`data:<mime>;base64,...`).

### Adding media from URLs

```python
dataset.add_media_url(row_index, "column_name", "https://example.com/img.jpg", media_type="image")
```

`media_type` must be one of: `image`, `audio`, `video`.

### Supported formats

| Type | Extensions |
|------|------------|
| Image | `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp` |
| Audio | `.mp3`, `.wav`, `.ogg`, `.m4a` |
| Video | `.mp4`, `.webm`, `.mov` |

### Using multimodal fields in tasks

```python
@ze.task(outputs=["diagnosis"])
def diagnose(row):
    result = call_vision_model(
        text=row.symptoms,
        image=row.chest_xray,  # data URI or URL string
    )
    return {"diagnosis": result}
```

## Schema and Column Types

Supported column types (inferred from Parquet Arrow metadata):

- `string` — text
- `int64` / `int32` — integer
- `float64` / `float32` — float
- `bool` — boolean

### Primary key detection order

1. Benchmark's configured primary key column
2. Column named `id`
3. First column ending in `_id`
4. First column

Include a stable `row_id` or `id` column for reliable resume and comparison.

## Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `ValueError: ZeroEval SDK not initialized` | `ze.init()` not called | Call `ze.init()` before any SDK operation |
| `ValueError: Must provide data` | Name given without `data=` | Pass `data=[...]` or a CSV file path |
| Dataset not found on pull | Wrong name or not pushed | Check the benchmark hub, verify org/slug |
| Subset not found | Subset name doesn't match a Parquet file | Check `data/*.parquet` filenames |
| Resume fails | Rows missing `row_id` | Add a `row_id` column to every row |
