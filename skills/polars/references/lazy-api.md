# Lazy API: Scans, Plans, Streaming

The lazy API builds a query plan that Polars optimizes before execution:
predicate pushdown (filters applied during the scan), projection pushdown
(only needed columns read), slice pushdown, common subexpression elimination,
and automatic parallelism — which is why queries start with `scan_*` and end
with a single `collect()`.

## Contents

Starting a lazy query; starting from an in-memory DataFrame; scan options
that prevent downstream pain; schema discovery without reading data;
inspecting the plan; executing; larger-than-memory streaming and sinks.

## Starting a lazy query

```python
lf = pl.scan_csv("data.csv")
lf = pl.scan_parquet("data.parquet")      # also: scan_ndjson, scan_ipc
lf = pl.scan_parquet("events/*.parquet")  # globs scan many files as one
lf = df.lazy()                            # from an eager DataFrame
```

## Starting from an in-memory DataFrame

Always call `.lazy()` on a `DataFrame` before building a query on it: eager
methods execute one at a time, with no plan to optimize. A function with a
`DataFrame -> DataFrame` signature keeps that signature — go lazy on entry,
collect on the way out:

```python
def top_regions(df: pl.DataFrame, n: int) -> pl.DataFrame:
    return (
        df.lazy()                                  # wrap
        .filter(pl.col("revenue") > 0)
        .group_by("region")
        .agg(pl.col("revenue").sum().alias("total"))
        .sort("total", descending=True)
        .head(n)
        .collect()                                 # unwrap
    )
```

If a step you need exists only on `DataFrame`, stay lazy up to that point,
`.collect()` there, and re-enter with `.lazy()` for the rest — do not drop
the whole chain to eager for one method.

## Scan options that prevent downstream pain

Handle dirty data at the scan so the rest of the query stays strict:

```python
lf = pl.scan_csv(
    "data.csv",
    null_values=["N/A", "NULL", ""],   # become real nulls, not strings
    try_parse_dates=True,              # ISO-like strings -> temporal dtypes
    infer_schema_length=10_000,        # sample more rows for dtype inference
    schema_overrides={"zipcode": pl.String},  # pin dtypes inference gets wrong
)
```

If a numeric column still arrives as String, something non-numeric is in it:
inspect with `lf.head(20).collect()`, then extend `null_values` or use
`.cast(pl.Float64, strict=False)` deliberately.

## Schema discovery without reading data

```python
lf.collect_schema()          # {name: dtype}, resolves the plan, no data read
lf.collect_schema().names()
lf.head(5).collect()         # tiny peek at actual values
```

Always run `collect_schema()` before executing a query: it validates the plan
and gives the real column names and dtypes without reading data, which is the
difference between one-shot success and a loop of `ColumnNotFoundError`.

## Inspecting the plan

```python
lf = pl.scan_csv("large.csv").select("a", "b").filter(pl.col("a") > 100)

print(lf.explain())                    # optimized plan as text
print(lf.explain(optimized=False))     # naive plan, for comparison
```

In the optimized plan, look for the filter inside the scan node (predicate
pushdown) and `PROJECT 2/47 COLUMNS` (projection pushdown). A predicate that
did not push down usually depends on a computed column — filter on source
columns where possible.

## Executing

```python
result = lf.collect()                  # optimize + run, returns DataFrame
```

Collect exactly once per query. For debugging a long chain, prefer
`lf.head(20).collect()` over collecting the full intermediate.

LazyFrames are cheap immutable values, so when several queries share a scan,
build them from one base and collect them together — common subplans are then
computed once. This is also the right shape for conversational analysis: keep
the base, extend it per follow-up question, collect once per answer.

```python
base = pl.scan_parquet("orders/*.parquet").filter(pl.col("year") == 2024)
by_region = base.group_by("region").agg(pl.col("rev").sum())
by_month = base.group_by(pl.col("date").dt.month()).agg(pl.col("rev").sum())
region_df, month_df = pl.collect_all([by_region, by_month])
```

## Larger-than-memory: streaming and sinks

```python
result = lf.collect(engine="streaming")    # process in batches
```

The streaming engine executes the plan in chunks, so datasets larger than
RAM still complete. For large outputs, skip materialization and sink straight
to disk:

```python
lf.sink_parquet("out.parquet")
lf.sink_csv("out.csv")
lf.sink_ndjson("out.ndjson")
```

Use streaming or sinks when the input is much larger than memory, or when a
regular `collect()` is killed by the OS.
