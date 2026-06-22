# From Question to Query: Insight Recipes

Query shapes for answering natural language questions about data. Each
recipe is a complete lazy chain: adapt column names, collect once.

Workflow reminder: discover the schema first (`lf.collect_schema()`,
`lf.head(5).collect()`), state how you interpreted vague terms ("top" =
by revenue, "last quarter" = 2026 Q1), then answer with numbers.

Throughout, `lf` is a LazyFrame, e.g.
`lf = pl.scan_csv("data.csv", try_parse_dates=True, null_values=["N/A", ""])`.

---

## "What are the top N X by Y?"

```python
(
    lf.group_by("product")
    .agg(pl.col("revenue").sum().alias("total_revenue"))
    .top_k(10, by="total_revenue")
    .sort("total_revenue", descending=True)
    .collect()
)
```

`top_k` avoids sorting the whole frame; the final small sort is for
presentation only.

## "Top N per group" (top 3 products per region)

```python
(
    lf.group_by("region", "product")
    .agg(pl.col("revenue").sum().alias("total"))
    .filter(
        pl.col("total").rank("dense", descending=True).over("region") <= 3
    )
    .sort("region", "total", descending=[False, True])
    .collect()
)
```

## "How did X change vs. the previous period?"

```python
(
    lf.group_by(pl.col("date").dt.truncate("1mo").alias("month"))
    .agg(pl.col("revenue").sum().alias("revenue"))
    .sort("month")
    .with_columns(
        prev=pl.col("revenue").shift(1),
    )
    .with_columns(
        growth_pct=(pl.col("revenue") / pl.col("prev") - 1) * 100,
    )
    .collect()
)
```

Per-group variant: use `.shift(1).over("region")` and sort by
`("region", "month")` first. State the definition used (month-over-month,
percent) in the answer.

## "What share does each X have of the total?"

```python
(
    lf.group_by("category")
    .agg(pl.col("revenue").sum().alias("revenue"))
    .with_columns(
        share_pct=pl.col("revenue") / pl.col("revenue").sum() * 100
    )
    .sort("share_pct", descending=True)
    .collect()
)
```

After `agg`, `pl.col("revenue").sum()` is the grand total; the division
broadcasts. The same trick works pre-aggregation with
`.sum().over("region")` for share-within-group.

## "What does the distribution of X look like?"

```python
# Five-number-style summary per group
(
    lf.group_by("segment")
    .agg(
        pl.col("order_value").min().alias("min"),
        pl.col("order_value").quantile(0.25).alias("p25"),
        pl.col("order_value").median().alias("median"),
        pl.col("order_value").quantile(0.75).alias("p75"),
        pl.col("order_value").max().alias("max"),
        pl.len().alias("n"),
    )
    .collect()
)

# Binning: fixed edges with cut, equal-sized buckets with qcut
(
    lf.with_columns(
        bucket=pl.col("order_value").qcut(4, labels=["q1", "q2", "q3", "q4"])
    )
    .group_by("bucket")
    .agg(pl.len().alias("n"), pl.col("order_value").mean().alias("avg"))
    .sort("bucket")
    .collect()
)
```

## "How does X trend over time?" (resampling)

```python
(
    lf.sort("ts")
    .group_by_dynamic("ts", every="1w")
    .agg(
        pl.col("revenue").sum().alias("revenue"),
        pl.col("order_id").n_unique().alias("orders"),
    )
    .collect()
)
```

`group_by_dynamic` requires a sorted time column. `every` accepts
durations like `"1d"`, `"1w"`, `"1mo"`, `"1q"`, `"1y"`; add `group_by=`
for one series per category.

## "Break X down by whether/which ..." (conditional breakdown)

```python
(
    lf.group_by("region")
    .agg(
        pl.col("amount").filter(pl.col("status") == "paid")
            .sum().alias("paid"),
        pl.col("amount").filter(pl.col("status") == "pending")
            .sum().alias("pending"),
        (pl.col("status") == "refunded").sum().alias("refund_count"),
    )
    .collect()
)
```

`expr.filter()` inside `agg` aggregates a subset per group. Summing a
boolean expression counts how often it is true.

## "Which X are above/below their group average?" (window)

```python
(
    lf.with_columns(
        dept_avg=pl.col("salary").mean().over("department"),
    )
    .filter(pl.col("salary") > pl.col("dept_avg"))
    .select("name", "department", "salary", "dept_avg")
    .collect()
)
```

## "How many distinct X ...?" / quick counts

```python
(
    lf.group_by("region")
    .agg(
        pl.len().alias("rows"),
        pl.col("customer_id").n_unique().alias("customers"),
        pl.col("order_id").null_count().alias("missing_orders"),
    )
    .collect()
)
```

`pl.len()` counts rows including nulls; `.count()` counts non-null values
of a column. Pick deliberately and say which you used if it matters.

---

## Sanity checks before presenting an answer

Cheap checks that catch most wrong answers:

```python
result = query.collect()

result.height                      # plausible number of rows?
result.null_count()                # unexpected nulls from a join or cast?
lf.select(pl.len()).collect()      # input row count for a gut check
```

If a join produced more rows than the left input, the key was not unique
on the right side. If an aggregate is null, the source column likely
contained only nulls after filtering, or a cast with `strict=False`
nulled unparsable values; check `lf.head().collect()` for dirty data and
move cleaning into the scan (`null_values=[...]`).
