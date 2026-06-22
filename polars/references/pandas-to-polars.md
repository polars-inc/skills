# Pandas → Polars Translation

## Contents

- General approach
- API patterns
- Parsing date/datetime strings
- Column renaming — `Expr.name` namespace
- Polars-specific correctness traps
- `pd.crosstab` → `.pivot()` with `aggregate_function="len"`
- Logic and algorithmic patterns

## General approach

Before writing any code:

1. **Understand the data**: inspect input schemas with `.collect_schema()` and clarify what the output shape should be.
2. **Plan the context chain**: decide which contexts are needed and in what order — `filter`, `select`, `with_columns`, `join`, `group_by`, `agg`, `sort`, `concat`, `explode`, `pivot`, `unpivot`, `transpose`, etc.
3. **Find the right expressions**: consult the Polars API reference for built-in expressions. Native expressions are always faster and clearer than custom code.
4. **Avoid Python-level row operations**: `map_elements`, `map_batches`, and `map_groups` are last-resort escape hatches with a 10–100× performance penalty. Only use them when no native expression exists.
5. **Use informative names**: column names and variables should clearly reflect their contents.
6. **Match the output column order**: pandas reorders columns implicitly (`reset_index()` moves the index column to the front; `df["new"] = ...` appends at the end). If the pipeline creates any new column, always finish with an explicit `.select(...)` listing the output columns in the exact order of the pandas result. See "Column order must match pandas exactly" below.

---

## API patterns

| Wrong | Correct | Notes |
|-------|---------|-------|
| `.method([a, b, c])` list form | `.method(a, b, c)` positional args | applies everywhere: `select`, `with_columns`, `group_by`, `agg`, `sort`, etc. |
| `pl.col("x").sum()` in `.agg()` | `pl.sum("x")` | top-level shorthand |
| `pl.col("x").mean()` in `.agg()` | `pl.mean("x")` | top-level shorthand |
| `(col >= a) & (col < b)` | `col.is_between(a, b, closed="left")` | for date ranges (half-open) |
| `(col >= a) & (col <= b)` | `col.is_between(a, b)` | for numeric ranges (closed) |
| No rounding on monetary/financial aggregations | `.round(2)` on revenue, price, balance columns | |
| `join_asof(..., tolerance=pl.duration(minutes=5))` | `join_asof(..., tolerance=timedelta(minutes=5))` | `tolerance` expects a Python `datetime.timedelta`, not a Polars duration expression |
| `pl.col("age").cut([0, 17, 24, 34, 44, 54, 120], labels=[...6...])` | `pl.col("age").cut([17, 24, 34, 44, 54], labels=[...6...])` | Polars `cut` takes only interior boundaries; `n` breaks → `n+1` bins/labels. Outer bounds are implicit. pandas `bins=` includes both outer bounds. |
| Column order drift after `with_columns`/`drop` | Finish with an explicit `.select(...)` in pandas order | see note above |

---

## Parsing date/datetime strings

pandas `pd.to_datetime(col)` infers formats. The Polars equivalent is `str.to_datetime()`, which also infers common formats including ISO datetimes.

| Wrong | Correct | Notes |
|-------|---------|-------|
| `str.to_date()` on datetime-shaped strings | `str.to_datetime()` | `str.to_date()` raises `ComputeError` unless the strings are pure dates like `"2018-01-01"` |
| `str.to_date()` to get day precision | `str.to_datetime().dt.truncate("1d")` (or `.dt.date()` for a `Date` dtype) | parse first, reduce precision second |
| pandas `format="…%f"` carried over verbatim | translate `%f` → `%.f` | pandas `%f` is microseconds; chrono `%f` is nanoseconds — `%.f` parses `.123456` correctly. All other common directives carry over verbatim. |

```python
# pandas — strings like "2018-01-01T00:00:00"
weather["date"] = pd.to_datetime(weather["date"])

# Wrong: ComputeError: could not find an appropriate format to parse dates
weather.with_columns(pl.col("date").str.to_date())

# Correct
weather.with_columns(pl.col("date").str.to_datetime())
```

---

## Column renaming — use `Expr.name` namespace instead of lambdas

Prefer the built-in `name` namespace over `rename()` with a lambda or dict comprehension.

| Pattern | Instead of | Use |
|---------|-----------|-----|
| Add prefix to all selected columns | `.rename({c: f"prefix_{c}" for c in cols})` | `pl.col(cols).name.prefix("prefix_")` |
| Add suffix to all selected columns | `.rename({c: f"{c}_suffix" for c in cols})` | `pl.col(cols).name.suffix("_suffix")` |
| Lowercase all column names | `df.rename(str.lower)` / lambda | `pl.col("*").name.to_lowercase()` (in `select`) |
| Uppercase all column names | lambda | `pl.col("*").name.to_uppercase()` |
| Rename with arbitrary function | `.rename(lambda c: fn(c))` | `pl.col(...).name.map(fn)` |
| Keep original name after expression | `.alias(original_name)` scattered | `pl.col(...).some_expr().name.keep()` |
| Add prefix to struct fields | manual struct manipulation | `pl.col("s").name.prefix_fields("prefix_")` |
| Add suffix to struct fields | manual struct manipulation | `pl.col("s").name.suffix_fields("_suffix")` |
| Rename struct fields with function | manual struct manipulation | `pl.col("s").name.map_fields(fn)` |

`name.prefix` / `name.suffix` / `name.to_lowercase` / `name.to_uppercase` are especially useful inside `.select()` or `.with_columns()` to bulk-rename without building a dict:

```python
# Instead of:
df.select([pl.col(c).alias(f"raw_{c}") for c in df.columns])

# Write:
df.select(pl.all().name.prefix("raw_"))
```

---

## Polars-specific correctness traps

| Wrong | Correct | Notes |
|-------|---------|-------|
| `float(bool_series.sum()) / len(df)` | `bool_series.sum() / df.height` | Polars division already returns float; use `.height` not `len()` for row count |
| `float(int_expr / int_expr)` | `int_expr / int_expr` | Polars strict typing: int ÷ int → float automatically, no `float()` cast needed |
| `round(row['col'], n)` inside `iter_rows()` loop | `.agg(pl.col(...).mean().round(n))` then `row['col']` directly | Python `round()` raises `TypeError` on `None`; round inside the expression instead |

---

## `pd.crosstab` → `.pivot()` with `aggregate_function="len"`

```python
# pandas
pd.crosstab(df["age_group"], df["activity_quartile"])

# Polars
df.pivot(
    on="activity_quartile",
    index="age_group",
    values="activity_quartile",
    aggregate_function="len",
    sort_columns=True,
)
```

Key details:
- Passing the **same column to `on` and `values`** works — no throwaway/sentinel column needed.
- `aggregate_function="len"` produces **zeros** for empty cells, not nulls — no `.fill_null(0)` needed (unlike `"first"`, `"sum"`, etc.).
- `sort_columns=True` gives deterministic column order (e.g. q1 → q2 → q3 → q4).
- `.pivot()` is available on **LazyFrames** as well as DataFrames.

---

## Logic and algorithmic patterns

| Scenario | Wrong | Correct |
|----------|-------|---------|
| SQL `WHERE key IN (SELECT ...)` / `EXISTS` | inner join + `.unique()` | `.join(subquery, on=key, how="semi")` |
| SQL `NOT IN` / `NOT EXISTS` | left join + null filter | `.join(subquery, on=key, how="anti")` — 1.5× faster |
| Multiple AND filter conditions | multiple chained `.filter()` calls | single `.filter(cond1 & cond2 & cond3)` or `.filter(cond1, cond2, cond3)` — positional args = AND |
| Multiple OR filter conditions | | single `.filter(cond1 \| cond2)` |
| Scalar threshold from a subquery | `.select(...).item()` → Python scalar | cross-join the subquery and compare column-to-column — stay lazy |
| Filter rows against an aggregate of the same frame | extract scalar with `.item()`, then filter | aggregations broadcast inside `filter`: `.filter(pl.col("x") > pl.col("x").quantile(0.9))` |
| Annotate rows with group-level aggregates (pandas: aggregate + merge back, or `transform`) | `group_by().agg()` + join back | window expression: `expr.over(keys)` inside `with_columns`; for time buckets `expr.over(pl.col(ts).dt.truncate("5m"))` |
| Dict lookup per value | `map_elements(lambda v: mapping[v])` | `.replace_strict(mapping)` — stays in the engine, 10–100× faster; raises on unmapped values like the lambda (pass `default=` to map misses instead) |
| `SUM(CASE WHEN cond THEN val ELSE 0 END)` | separate `_tmp` column + drop | inline: `pl.when(cond).then(val).otherwise(0).sum()` inside `.agg()` |
| Same table joined twice (e.g. nation as supplier_nation and customer_nation) | filter into two frames + `pl.concat` | filter to `.is_in([v1, v2])`, join the frame twice with column aliases, then filter valid direction combos |
