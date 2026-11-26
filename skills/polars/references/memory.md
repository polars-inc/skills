# Memory Limits and Out-of-Core Processing

`collect()` runs on the streaming engine, which processes data in batches
and spills to disk when it nears its memory budget, so most queries larger
than RAM just work. When one fails anyway, find out why with the checklist
below rather than rewriting the query as eager code or manual chunk loops.
Both of those use more memory, not less.

## Contents

The memory budget; configuring it; operations that cannot stream;
diagnosing out-of-memory; on a cluster.

## The memory budget

- The budget is `POLARS_OOC_MEMORY_BUDGET_FRACTION` (default `0.8`) times
  total memory, capped by `POLARS_OOC_MEMORY_BUDGET_MB` if set. MB means
  1,000,000 bytes.
- Total memory is the smallest of system RAM and any cgroup limit, so a
  container's limit is respected.
- Above the budget, sorts, group-bys, equi-joins (including semi and anti
  joins), `over()` windows, `unique()` and the final `collect()` buffer
  spill to disk and read back when needed. Results are identical, only
  slower.
- The budget counts only memory Polars allocates. Python objects, NumPy
  arrays and other libraries in the same process take from the remaining
  20%.
- `engine="in-memory"` never spills. Do not pass it for large data.

## Configuring it

All these are environment variables **read when `polars` is imported**.
Setting `os.environ` after `import polars` has no effect, so set them in
the shell, the container spec, or at the top of the entry script before
the import:

```python
import os
os.environ["POLARS_OOC_MEMORY_BUDGET_MB"] = "8000"    # before importing polars
import polars as pl
```

| Variable | Default | Purpose |
|---|---|---|
| `POLARS_OOC_MEMORY_BUDGET_FRACTION` | `0.8` | Fraction of total memory Polars may use before spilling |
| `POLARS_OOC_MEMORY_BUDGET_MB` | unlimited | Absolute cap; the budget is the smaller of the two |
| `POLARS_OOC_SPILL_DIR` | `/var/tmp/polars-$USER/spill` (Linux), temp dir elsewhere | Where spill files go; use a fast local disk, not tmpfs |
| `POLARS_OOC_DISK_BUDGET_MB` | `64000` | Max spilled data before the query aborts |
| `POLARS_OOC_SPILL_COMPRESSION_LEVEL` | `1` | zstd level for spill files; `0` disables compression |
| `POLARS_OOC_LOG_METRICS` | off | `1` prints per-operator spill stats, to confirm spilling |

Spill files are deleted when the query no longer needs them.

## Operations that cannot stream

Some operations load their whole input into memory and run it on the
in-memory engine. A fallback is the usual reason a streaming query runs
out of memory. Common ones:

- `map_batches` and other Python UDFs over whole columns
- `median`, `quantile`, `rank`
- joins with `validate=`
- `join_asof(by=...)` on unsorted `by` columns
- `unique(keep="last", maintain_order=True)`

Find fallbacks in the physical plan. The nodes `in-memory-map`,
`in-memory-join` and `columnar-function` mark them:

```python
dot = lf.show_graph(engine="streaming", plan_stage="physical", raw_output=True)
print([n for n in ("in-memory-map", "in-memory-join", "columnar-function") if n in dot])
```

`lf.explain()` shows the logical plan only; it does not reveal fallbacks.
Fix a fallback by filtering and selecting before it, so less data reaches
it, or by replacing a UDF with a native expression.

## Diagnosing out-of-memory

Work through these in order:

1. **Collecting a huge result?** The output of `collect()` must fit in RAM.
   Write large outputs with `lf.sink_parquet(...)` (or `sink_csv`,
   `sink_ipc`, `sink_ndjson`) instead.
2. **Eager code or `engine="in-memory"`?** Make it one lazy query on the
   default engine (SKILL.md "Default stance"). `read_*` loads whole files,
   so use `scan_*`.
3. **Fallback in the plan?** Check with `show_graph` as above.
4. **Is the detected limit right?** Run with `POLARS_VERBOSE=1` and check
   the `total memory:` line against the machine or container limit.
5. **Is it spilling?** Run with `POLARS_OOC_LOG_METRICS=1`. No spill lines
   while memory is exhausted means something outside the budget is using
   it: Python objects, other libraries, or other processes. Lower
   `POLARS_OOC_MEMORY_BUDGET_FRACTION` or set `POLARS_OOC_MEMORY_BUDGET_MB`
   to leave headroom.
6. **`ComputeError: ... exceeded the out-of-core disk budget`?** Raise
   `POLARS_OOC_DISK_BUDGET_MB` or point `POLARS_OOC_SPILL_DIR` at a larger
   disk.
7. **Still too big or too slow for one machine?** Run the same query on a
   cluster with Polars Nanook (`references/nanook.md`).

Filtering early and selecting only the needed columns, ideally in the scan,
reduces memory more than any setting.

## On a cluster

Nanook workers spill the same way. Set the memory per worker with the
deployment's memory limit, and fix out-of-memory with more partitions or
workers: see "Memory" in `references/nanook.md`.
