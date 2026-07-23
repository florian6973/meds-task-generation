# MEDS Random Task Sampler

[![Python 3.11+](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Model-independent generation of query-based task rows from MEDS datasets.

The package provides two separate workflows:

- `random_sample`: random `(code, duration)` specifications paired with random patient contexts; and
- `dense_grid`: explicit `code x duration` grids at sampled patient prediction times.

These names describe how rows are sampled, not how a downstream model must use them. For example, either output
could be used for training, validation, benchmarking, probing, or analysis.

Both workflows follow
[`payalchandak/EveryQuery@9bd85a1`](https://github.com/payalchandak/EveryQuery/commit/9bd85a1d2c68000aa9362731c7612007d262ac56).
The package owns the shared task schema, code-source resolution, future-occurrence labeling, death and censoring
semantics, deterministic seeds, and atomic output writes. It does not depend on Hydra or a model framework.

## Random task samples

```python
from meds_random_task_sampler import RandomTaskSamplerConfig, sample_random_tasks

config = RandomTaskSamplerConfig(
    num_queries=1024,
    num_contexts_per_query=1,
    min_prediction_times_per_subject=50,
    query_codes="/path/to/MEDS",  # resolves metadata/codes.parquet
    min_duration=1,
    max_duration=731,
    duration_distribution="log-uniform",
)

result = sample_random_tasks(
    data_dir="/path/to/MEDS",
    output_dir="/path/to/random_tasks",
    split="train",
    config=config,
)
```

Output is partitioned under `random_tasks/{split}/*.parquet`; restartable intermediate artifacts use the sibling
`random_tasks_artifacts/{split}/` directory. Machine-readable summary statistics are written to
`random_tasks_artifacts/{split}/_summary.json`.

## Dense task grids

```python
from meds_random_task_sampler import (
    TaskGridGeneratorConfig,
    generate_task_grid,
)

config = TaskGridGeneratorConfig(
    prediction_times_per_subject=1,
    min_context_per_subject=50,
    query_codes=["CODE_A", "CODE_B"],
    durations=[30, 90, 180, 365, 731],
    write_unique_prediction_times=True,
    censored_rows="keep",  # or "drop" for current EveryQuery evaluation behavior
)

result = generate_task_grid(
    data_dir="/path/to/MEDS",
    output_dir="/path/to/task_grid",
    split="held_out",
    input_shard="0",
    config=config,
)
```

Grid rows are written to `task_grid/{split}/{shard}.parquet`. Optional unique prediction times use the sibling
`task_grid_unique/` root and per-shard summaries use `task_grid_summary/`. Nullable/censored labels are retained
by default; use `censored_rows="drop"` to reproduce current EveryQuery evaluation output.

See [DESIGN.md](DESIGN.md) for the behavioral contract and planned EveryQuery adapter boundary.

## Development

```bash
uv sync --group dev
uv run pytest -v
uv run pre-commit run --all-files
```

This repository retains the
[`McDermottHealthAI/MHAL-template`](https://github.com/McDermottHealthAI/MHAL-template) project structure.
