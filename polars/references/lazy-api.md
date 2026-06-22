# Lazy API: Scans, Plans, Streaming

The lazy API builds a query plan that Polars optimizes before execution:
predicate pushdown (filters applied during the scan), projection pushdown
(only needed columns read), slice pushdown, common subexpression
elimination, and automatic parallelism. This is why queries start with
`scan_*` and end with a single `collect()`.

## Contents

- Starting a lazy query
- Scan options that prevent downstream pain
- Schema discovery without reading data
- Inspecting the plan
- Executing
- Larger-than-memory: streaming and sinks
- Composition pattern

## Starting a lazy query

```python
lf = pl.scan_csv("data.csv")
lf = pl.scan_parquet("data.parquet")      # also: scan_ndjson, scan_ipc
lf = pl.scan_parquet("events/*.parquet")  # globs scan many files as one
lf = df.lazy()                            # from an eager DataFrame
```

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

If a numeric column still arrives as String, something non-numeric is in
it; inspect with `lf.head(20).collect()` and either extend `null_values`
or use `.cast(pl.Float64, strict=False)` deliberately.

## Schema discovery without reading data

```python
lf.collect_schema()          # {name: dtype}, resolves the plan, no data read
lf.collect_schema().names()
lf.head(5).collect()         # tiny peek at actual values
```

Always check the schema before writing expressions against unfamiliar
data; it is the difference between one-shot success and an iteration
loop on `ColumnNotFoundError`.

## Inspecting the plan

```python
lf = pl.scan_csv("large.csv").select("a", "b").filter(pl.col("a") > 100)

print(lf.explain())                    # optimized plan as text
print(lf.explain(optimized=False))     # naive plan, for comparison
```

In the optimized plan, look for the filter inside the scan node
(predicate pushdown) and `PROJECT 2/47 COLUMNS` (projection pushdown).
If a predicate did not push down, it usually depends on a computed
column; filter on source columns where possible.

## Executing

```python
result = lf.collect()                  # optimize + run, returns DataFrame
```

Collect exactly once per query. An intermediate `collect()` materializes
everything and discards the plan, so later steps optimize from scratch.
For debugging a long chain, prefer `lf.head(20).collect()` over
collecting the full intermediate.

When the same scan feeds several final queries, build them as separate
LazyFrames from one shared base and collect them together so common
subplans are computed once:

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

The streaming engine executes the plan in chunks so datasets larger than
RAM still complete. For large outputs, skip materialization entirely and
sink straight to disk:

```python
lf.sink_parquet("out.parquet")
lf.sink_csv("out.csv")
lf.sink_ndjson("out.ndjson")
```

Use streaming or sinks when the input is much larger than memory or when
a regular `collect()` is killed by the OS. Aggregated results small
enough to inspect can still be collected normally afterward by scanning
the sink output.

## Composition pattern

LazyFrames are cheap immutable values; build complex queries from a
shared base:

```python
data = pl.scan_csv("data.csv")
active = data.filter(pl.col("status") == "active")

summary = (
    active.group_by("category")
    .agg(
        pl.col("amount").sum().alias("total"),
        pl.col("amount").mean().alias("average"),
    )
    .collect()
)
```

This is also the right shape for conversational analysis: keep the base
LazyFrame, answer each follow-up question by extending it, collect once
per answer.
