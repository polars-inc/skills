# Expression Contexts

Expressions are lazy descriptions of transformations. They compute nothing
until placed in a **context**; the context determines what columns appear
in the output, how broadcasting works, and the row count of the result.

## Contents

`select()`, `with_columns()`, `filter()`, `group_by() + agg()`, `over()`,
`sort()`, `join()`, putting contexts together.

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

Expression expansion works here too — one expression covering many columns,
expanded against the schema and run in parallel: `lf.select(cs.numeric())`,
`lf.select((cs.float() * 1.1).name.suffix("_adj"))`. Selection forms, set
operations and renaming: see the expansion section of `expressions.md`.

## with_columns()

Adds or replaces columns; everything else passes through. A plain
aggregation like `pl.col("v").mean()` produces a scalar that Polars
broadcasts to every row — it does not error. Use `over("group")` for the
*per-group* aggregate aligned to each row instead.

```python
lf.with_columns(
    (pl.col("quantity") * pl.col("price")).alias("total"),
    pl.col("price").cast(pl.Float64),                # replaces "price"
    pl.col("date").dt.year().alias("year"),
    is_active=(pl.col("status") == "active"),        # kwarg = alias
)
```

One context, one expression per distinct operation. When the operation is
the same for every column, expand instead of generating one expression per
column:

```python
# avoid: one expression per column (a loop of contexts is worse still)
lf.with_columns((pl.col(c) * 2).alias(f"{c}_scaled") for c in ["a", "b", "c"])

# prefer: one expanded expression
lf.with_columns((pl.col("a", "b", "c") * 2).name.suffix("_scaled"))

# a comprehension is fine when the logic genuinely differs per column
lf.with_columns((pl.col(c) * m).alias(f"{c}_s") for c, m in factors.items())
```

## filter()

Keeps rows where the boolean expression is true; combine conditions with
`&`, `|`, `~`, each in parentheses.

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

Null behavior: `filter(pl.col("v") > 10)` silently drops null rows, because
a comparison against null is null; add `| pl.col("v").is_null()` to keep them.

## group_by() + agg()

One output row per unique group; output columns are the grouping keys plus
the aggregations. `maintain_order=True` keeps first-seen group order (costs
some parallelism).

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

Conditional aggregation, the workhorse for breakdown questions —
`expr.filter()` aggregates a subset per group, and summing a boolean counts
how often it is true:

```python
pl.col("salary").filter(pl.col("level") == "senior").mean()
(pl.col("status") == "active").sum()
```

Sorting within a group to get "the X with the highest Y":

```python
lf.group_by("category").agg(
    pl.col("product").sort_by("revenue", descending=True)
        .first().alias("top_product")
)
```

To control `first()`/`last()` semantics, sort the frame before grouping:
`lf.sort("date", descending=True).group_by("customer").agg(...)`.

## over() - window functions

Computes a grouped result inside `select()` or `with_columns()` while
keeping the original row count — the way to put a group aggregate on every
row.

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

`lf.group_by("g").agg(pl.col("v").mean())` returns one row per group;
`lf.select(pl.col("v").mean().over("g"))` returns the input row count with
the group mean repeated.

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

- A left join introduces nulls for unmatched rows; check
  `result.null_count()` after joining.
- If the right key is not unique, the left side fans out: more rows after
  an inner/left join means duplicate keys on the right.
- Anti joins are the idiomatic "which X never did Y".

---

## Putting contexts together

Same order as the canonical pattern in `SKILL.md`; note the second
`with_columns()`, which computes from the aggregates:

```python
result = (
    lf
    .filter((pl.col("year") == 2024) & (pl.col("status") == "completed"))
    .group_by("region", "month")
    .agg(
        pl.col("profit").sum().alias("total_profit"),
        pl.col("customer_id").n_unique().alias("customers"),
    )
    .with_columns(
        per_customer=pl.col("total_profit") / pl.col("customers")
    )
    .sort("total_profit", descending=True)
    .collect()
)
```

Anti-patterns:

```python
# filtering after expensive work
lf.group_by("id").agg(...).filter(pl.col("year") == 2024)   # avoid
lf.filter(pl.col("year") == 2024).group_by("id").agg(...)   # prefer

# loop of contexts
for c in cols: lf = lf.with_columns(pl.col(c) * 2)     # avoid
lf.with_columns(pl.col(cols) * 2)                      # prefer, replaces cols
lf.with_columns((pl.col(cols) * 2).name.suffix("_x"))  # prefer, adds new cols
```
