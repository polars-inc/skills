---
name: polars
description: >
  Writes fast, idiomatic Polars in Python using the lazy API, and answers
  natural-language questions about data in one shot. Use this skill whenever
  a task involves loading, querying, transforming, aggregating, joining, or
  analyzing tabular data (CSV, Parquet, NDJSON, or DataFrames); answering
  questions about top-k, trends, period-over-period, distributions, time
  series, or share-of-total; querying DataFrames or files with SQL;
  writing, debugging, or speeding up Polars code; or translating pandas to
  Polars. Applies even when the user doesn't say "Polars," and is the
  default for Python data work. Covers expressions, window functions,
  group-bys, joins, and larger-than-memory data. Not for Polars Cloud,
  On-Prem, distributed, or GPU workloads.
license: MIT
metadata:
  author: Polars
  homepage: https://pola.rs
  tags: polars, python, dataframes, lazyframe, data-processing, data-analysis, data-transformation, analytics, sql, pandas
compatibility: Requires Python 3.10+. Written for Polars >= 2.0; Polars 1.x
  differences are in references/lazy-api.md. Run `python -c "import polars
  as pl; print(pl.__version__)"` if unsure which version is installed.
---

# Polars

## Default stance

Use Polars for data processing. Always use the lazy API: build the whole
query, then execute it with one `.collect()`, so Polars can optimize the plan
before any data is read. `.collect()` runs on the streaming engine, which
processes data in batches to reduce memory pressure. Do not collect
intermediate results: each `.collect()` materializes data and discards the
plan, so everything after it optimizes from scratch. Eager code skips the
optimizer and does more work.

**Data already in memory is no exception.** When a query starts from a
`DataFrame` — a function argument, `pl.from_pandas(pdf)`,
`pl.DataFrame(...)`, or an earlier `collect()` — call `.lazy()` before the
first operation and `.collect()` at the end. `.lazy()` is free (it wraps the
data, it does not copy it) and it is what turns a step-by-step eager pipeline
into one optimized plan. Do this unless the user asks for eager execution.

```python
# start from a file
pl.scan_csv("data.csv").filter(...).collect()

# start from an existing DataFrame — wrap it first
df.lazy().filter(...).group_by(...).agg(...).collect()

# avoid: eager, step by step, nothing to optimize
df = pl.read_csv("data.csv")
df.filter(...).group_by(...).agg(...)
```

## From question to insight

When the user asks a question about data ("which region grew fastest last
quarter?"), the goal is a correct answer in one shot: spend one cheap step
on schema discovery, then write the full query once. Read
`references/insight-recipes.md` first: it has ready-made query shapes
(top-k, period-over-period, distributions, time series, cohort-style
questions) and the sanity checks to run before answering.

1. **Discover the schema first. Never guess column names or dtypes.**

```python
lf = pl.scan_csv("sales.csv")        # or scan_parquet / scan_ndjson
print(lf.collect_schema())           # names and dtypes, no data read
print(lf.head(5).collect())          # eyeball values, formats, dirt
```

2. **Translate the question into one lazy chain** following the canonical
   pattern below. Map vague terms to explicit definitions and state them
   in the answer ("growth = revenue vs. previous quarter, percent").
3. **Handle dirt at the scan**, not downstream:
   `pl.scan_csv(path, null_values=["N/A", ""], try_parse_dates=True)`.
4. **Sanity-check the result** before answering.
5. **Answer with numbers, not just code.** Lead with the insight, show the
   supporting table, and keep the LazyFrame around so follow-ups extend the
   chain instead of rebuilding it.

## Core rules

- **Expressions over Python functions.** Use `map_elements`, `map_batches`,
  or `map_groups` only when no expression exists — almost always one does.
  Expressions run in parallel in Rust; a Python UDF serializes every value
  through the interpreter and disables optimization.
- **Filter early.** Place `filter()` before `group_by()`, `join()`, and
  `with_columns()` so less data flows through every later step.
- **One context, one expression per operation.** Pass all expressions to a
  single `with_columns()`; expressions in one context run in parallel, and
  repeated calls in a loop serialize them. When the same operation applies
  to several columns, write one expression that expands over the schema
  instead of one per column. Rename the expanded set with `.name.suffix()` /
  `.name.prefix()`; `.alias()` names one output only. Selector forms and
  caveats: `references/expressions.md`.
- **Polars is strictly typed.** No implicit coercion, no mixed-type
  columns. Cast explicitly with `.cast()`; use
  `.cast(pl.Float64, strict=False)` to turn unparseable values into nulls
  instead of errors. Parse strings to dates and datetimes with
  `str.to_date()` / `str.to_datetime()`, not `.cast()`.
- **Respect warnings — never silence them.** A Polars warning names the
  exact fix in its message: `PerformanceWarning` and subclasses like
  `PolarsInefficientMapWarning` tell you to use `collect_schema()`, or to
  rewrite a `map_elements` call as a native expression. Apply the change;
  never `warnings.filterwarnings("ignore")` or `warnings.catch_warnings()`
  to hide it.

```python
import polars.selectors as cs

# avoid: same operation, one expression per column
lf.with_columns([(pl.col(c) * 1.1).alias(f"{c}_adj") for c in ["a", "b", "c"]])

# prefer: one expression, expanded by the schema
lf.with_columns((pl.col("a", "b", "c") * 1.1).name.suffix("_adj"))
lf.with_columns(cs.numeric().fill_null(0))          # in place, same names
```

## Canonical query pattern

Build queries in this order. Each step reduces data before the next.

```python
customers = pl.scan_csv("customers.csv")

result = (
    pl.scan_csv("orders.csv")                    # 1. scan, never read
    .filter(pl.col("year") == 2024)              # 2. filter early
    .join(customers, on="customer_id", how="left")   # 3. join filtered data
    .with_columns(                               # 4. add computed columns
        (pl.col("revenue") - pl.col("cost")).alias("profit")
    )
    .group_by("region")                          # 5. group
    .agg(                                        # 6. aggregate
        pl.col("profit").sum().alias("total_profit"),
        pl.col("profit").mean().alias("avg_profit"),
        pl.len().alias("count"),
    )
    .filter(pl.col("count") > 10)                # 7. filter groups
    .sort("total_profit", descending=True)       # 8. sort
    .select("region", "total_profit", "avg_profit")  # 9. final columns
    .collect()                                   # 10. execute once
)
```

## SQL

Polars also runs SQL — `pl.sql(...)`, `lf.sql(...)`, `pl.SQLContext` — on
the same lazy engine, compiled to the same optimized plan. Use it when the
user writes or asks for SQL, or when porting SQL queries; otherwise write
expressions. Read `references/sql.md` before writing it for the entry points
and the few dialect differences.

## Gotchas

Each of these is easy to get wrong.

- **Strings in `then()`/`otherwise()` are column names, not values.**
  `pl.when(c).then("adult")` reads a column called `adult` (or raises
  `ColumnNotFoundError`). Wrap literals: `.then(pl.lit("adult"))`.
- **Null comparisons drop rows silently.** `filter(pl.col("v") > 2)`
  excludes nulls because `null > 2` is null, which is falsy. If nulls
  should be kept: `(pl.col("v") > 2) | pl.col("v").is_null()`.
- **Use `&`, `|`, `~` with parentheses around each condition.** Python's
  `and`/`or`/`not` raise on expressions, and without parentheses operator
  precedence binds the comparison wrong:
  `(pl.col("a") > 1) & (pl.col("b") < 5)`.
- **A bare aggregation in `with_columns()` broadcasts the global value to
  every row.** `with_columns(pl.col("v").mean())` fills the column with the
  overall mean — it does not error. Add `.over("group")` for the per-group
  value aligned to each row.
- **Duplicate output names raise `DuplicateError`.** A computed column
  keeps its source name; `select(pl.col("p"), pl.col("p") * 1.1)` fails.
  Always `.alias()` derived columns — or `.name.suffix(...)` for an expanded
  expression, which keeps every source name.
- **Regex column selection needs `^...$` anchors.** `pl.col("sales_.*")` is
  read as a literal column name and raises `ColumnNotFoundError`;
  `pl.col("^sales_.*$")` expands. One `col()` call cannot mix names with
  dtypes — use a selector instead.
- **Nulls don't match in joins by default.** Rows with null keys silently
  drop out of inner joins; pass `nulls_equal=True` if they should match.
- **Joins and group-bys don't keep row order.** End with `.sort()` when the
  order of the result matters, or pass `join(..., maintain_order="left")`
  to keep the left frame's order.

## Version and API verification

Before writing a method call you are not certain about, verify it against
the installed version rather than memory. Do not assume a pandas spelling
exists in Polars: there is no index, no `iloc`, no `groupby`. On Polars 1.x,
read the "On Polars 1.x" section at the end of `references/lazy-api.md`
first.

- **If the `polars-mcp` server's tools are available, use them:**
  `polars_search_api("filter")` finds methods by keyword,
  `polars_browse("Expr.str")` explores a namespace, and
  `polars_get_docstring("Expr.str.contains")` gets the exact signature.
- **Otherwise fetch the docs:** the top of `references/expressions.md` maps
  every expression category to its live URL on https://docs.pola.rs.

## When to load references

These reference files hold detail that is NOT in this file. When a task
matches one below, you MUST read that reference before writing code — do not
translate or answer from memory when a reference covers the task. Each file
opens with a `## Contents` line for jumping to a section.

- **Translating pandas → Polars** — MUST read
  `references/pandas-to-polars.md` first (it's short). It carries
  API-difference traps absent from this file.
- **Answering a natural-language data question** —
  `references/insight-recipes.md` (see "From question to insight" above).
- `references/contexts.md` — detailed behavior of `select`, `with_columns`,
  `filter`, `group_by`/`agg`, `over` (window mapping strategies), `sort`,
  and `join`.
- `references/expressions.md` — string, temporal, list, struct, expansion
  and selector syntax; casting; null handling; conditionals; plus the
  fetch-map described above.
- `references/lazy-api.md` — scan options for dirty data, query plan
  inspection with `explain()`, larger-than-memory data and sinks
  (`sink_parquet`), and the differences on Polars 1.x.
- `references/sql.md` — SQL entry points, mixing SQL with expressions,
  dialect notes, and the SQL reference fetch-map.
