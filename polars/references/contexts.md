# Expression Contexts

Expressions are lazy descriptions of transformations. They compute nothing
until placed in a **context**; the context determines what columns appear
in the output, how broadcasting works, and the row count of the result.

| Context | Purpose | Keeps other columns | Row count |
|---|---|---|---|
| `select()` | Project/transform columns | No | Same (or 1 for aggregates) |
| `with_columns()` | Add/replace columns | Yes | Same |
| `filter()` | Subset rows | Yes | Fewer |
| `group_by().agg()` | Aggregate per group | No | One per group |
| `over()` | Window within another context | Yes | Same |
| `sort()` | Reorder rows | Yes | Same |
| `join()` | Combine two frames | Both sides | Depends on `how` |

---

## select()

Returns only the specified columns. Expressions must produce series of the
same length or scalars; scalars broadcast.

```python
lf.select(
    pl.col("name"),
    (pl.col("revenue") - pl.col("cost")).alias("profit"),
    pl.col("age").mean().alias("avg_age"),   # scalar, broadcast
    pl.lit(25).alias("target"),              # literal, broadcast
)
```

Expression expansion applies one expression to many columns; expanded
expressions run in parallel:

```python
import polars.selectors as cs

lf.select(cs.numeric())                          # all numeric columns
lf.select(pl.all().exclude("id", "created_at"))  # everything but
lf.select(pl.col("^sales_.*$"))                  # regex (^...$ required)
lf.select(pl.col("height", "weight").mean())     # expands to 2 exprs
lf.select((cs.float() * 1.1).name.suffix("_adj"))
```

Selectors compose with set operations: `cs.numeric() | cs.temporal()`,
`cs.numeric() - cs.ends_with("_id")`, `cs.contains("temp")`,
`cs.starts_with("sales_")`, `cs.matches(r"^\d+$")`.

Renaming: `.alias("x")` for one column; `.name.prefix("y_")`,
`.name.suffix("_z")`, `.name.map(str.upper)` for expanded expressions.

Computed columns keep their source name. Two outputs with the same name
raise `DuplicateError`; always alias derived columns.

## with_columns()

Adds or replaces columns; everything else passes through. A plain
aggregation like `pl.col("v").mean()` produces a scalar that Polars
broadcasts to every row — it does not error. Use `over("group")` when
you want the *per-group* aggregate aligned to each row instead of the
global value.

```python
lf.with_columns(
    (pl.col("quantity") * pl.col("price")).alias("total"),
    pl.col("price").cast(pl.Float64),                # replaces "price"
    pl.col("date").dt.year().alias("year"),
    is_active=(pl.col("status") == "active"),        # kwarg = alias
)
```

Generate expressions programmatically, then pass them in one call:

```python
# avoid: loop of with_columns calls, each is a separate context
# prefer: one call with a comprehension
lf.with_columns(
    (pl.col(c) * 2).alias(f"{c}_scaled") for c in ["a", "b", "c"]
)
```

## filter()

Keeps rows where the boolean expression is true. Combine conditions with
`&`, `|`, `~`, each wrapped in parentheses. Python `and`/`or`/`not` raise.

```python
lf.filter(
    (pl.col("age") >= 18)
    & pl.col("status").is_in(["active", "pending"])
    & pl.col("date").is_between(start, end)
    & ~pl.col("is_deleted")
)
```

Common predicates: `is_in`, `is_between` (inclusive), `is_null`,
`is_not_null`, `str.contains`, `str.starts_with`, `str.ends_with`.

Null behavior: a comparison against null yields null, which filter treats
as false. `filter(pl.col("v") > 10)` silently drops null rows; add
`| pl.col("v").is_null()` to keep them.

## group_by() + agg()

One output row per unique group. Output columns are the grouping keys plus
the aggregations. Use `maintain_order=True` to keep first-seen group order
(costs some parallelism).

```python
lf.group_by("region", "channel").agg(
    pl.col("revenue").sum().alias("revenue"),
    pl.col("revenue").mean().alias("avg_order"),
    pl.col("customer_id").n_unique().alias("customers"),
    pl.len().alias("rows"),
)
```

Grouping keys can be expressions:

```python
lf.group_by(
    (pl.col("date").dt.year() // 10 * 10).alias("decade"),
    (pl.col("height") < 1.7).alias("is_short"),
).agg(pl.len())
```

Aggregation building blocks:

```python
pl.col("v").sum() / .mean() / .median() / .std() / .min() / .max()
pl.col("v").quantile(0.95)          # e.g. 0.95 = 95th percentile
pl.len()                            # rows in group, nulls included
pl.col("v").count()                 # non-null values
pl.col("v").null_count()            # null values in group
pl.col("v").n_unique()              # distinct values in group
pl.col("v").first() / .last()       # order = current row order within group
pl.col("v").implode()           # collect group values into a list
```

Conditional aggregation, the workhorse for breakdown questions:

```python
lf.group_by("department").agg(
    pl.col("salary").filter(pl.col("level") == "senior")
        .mean().alias("avg_senior_salary"),
    (pl.col("status") == "active").sum().alias("active_count"),
)
```

Sorting within a group to get "the X with the highest Y":

```python
lf.group_by("category").agg(
    pl.col("product").sort_by("revenue", descending=True)
        .first().alias("top_product")
)
```

To control `first()`/`last()` semantics, sort the frame before grouping:
`lf.sort("date", descending=True).group_by("customer").agg(
pl.col("order_id").first().alias("latest_order"))`.

## over() - window functions

Computes a grouped result inside `select()` or `with_columns()` while
keeping the original row count. This is the way to put a group aggregate
on every row.

```python
lf.with_columns(
    dept_avg=pl.col("salary").mean().over("department"),
    rank_in_cat=pl.col("score").rank("dense", descending=True)
        .over("category"),
    running=pl.col("amount").cum_sum().over("customer_id"),
    prev_price=pl.col("price").shift(1).over("product_id"),
    city_total=pl.col("sales").sum().over("country", "city"),
)
```

`group_by` vs `over`: `lf.group_by("g").agg(pl.col("v").mean())` returns
one row per group; `lf.select(pl.col("v").mean().over("g"))` returns the
input row count with the group mean repeated.

Mapping strategies control how non-scalar window results map back:

- `"group_to_rows"` (default): each value returns to its original row.
  Requires the result to have the same length as the group.
- `"join"`: the whole group result becomes a list, repeated on every row
  of the group.
- `"explode"`: rows are emitted grouped together; faster, but row order
  changes. Useful for top-n-per-group projections.

```python
lf.select(
    pl.col("athlete").sort_by("rank")
        .over("country", mapping_strategy="explode")
)
```

## sort()

```python
lf.sort("date", descending=True)
lf.sort("department", "salary", descending=[False, True])
lf.sort(pl.col("revenue") / pl.col("cost"), descending=True)
lf.sort("value", nulls_last=True)
```

## join()

```python
lf.join(other, on="id", how="inner")
lf.join(other, on=["year", "month"], how="left")
lf.join(other, left_on="id", right_on="user_id", how="left")
lf.join(other, on="id", how="anti")    # rows in lf with no match
lf.join(other, on="id", how="semi")    # rows in lf with a match, lf cols only
```

Join facts that bite:

- Null keys never match by default; pass `nulls_equal=True` if two null
  keys should be considered equal.
- A left join introduces nulls for unmatched rows; check
  `result.null_count()` after joining.
- If the right key is not unique, the left side fans out: more rows after
  an inner/left join means duplicate keys on the right.
- Anti joins are the idiomatic "which X never did Y".

---

## Putting contexts together

```python
result = (
    lf
    .filter((pl.col("year") == 2024) & (pl.col("status") == "completed"))
    .with_columns(
        profit=pl.col("revenue") - pl.col("cost"),
        month=pl.col("date").dt.month(),
    )
    .group_by("region", "month")
    .agg(
        pl.col("profit").sum().alias("total_profit"),
        pl.col("customer_id").n_unique().alias("customers"),
    )
    .with_columns(
        per_customer=pl.col("total_profit") / pl.col("customers")
    )
    .sort("total_profit", descending=True)
    .select("region", "month", "total_profit", "per_customer")
    .collect()
)
```

Anti-patterns:

```python
# Python UDF: serial, no optimization
lf.select(pl.col("v").map_elements(lambda x: x * 2))   # avoid
lf.select(pl.col("v") * 2)                             # prefer

# filtering after expensive work
lf.group_by("id").agg(...).filter(pl.col("year") == 2024)   # avoid
lf.filter(pl.col("year") == 2024).group_by("id").agg(...)   # prefer

# loop of contexts
for c in cols: lf = lf.with_columns(pl.col(c) * 2)     # avoid
lf.with_columns(pl.col(*cols) * 2)                     # prefer
```
