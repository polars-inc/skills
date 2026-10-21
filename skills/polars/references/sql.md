# SQL Interface

Polars runs SQL on the same lazy engine as expressions: a query compiles to
the same optimized plan, so it is as fast as the equivalent expression chain.
Use SQL when the user writes or asks for SQL, or when porting SQL queries;
otherwise write expressions.

## Contents

Entry points; mixing SQL and expressions; dialect notes; SQL reference map.

## Entry points

Each returns a `LazyFrame`: build the query, then `.collect()` once. Load
files with `pl.scan_*` and reference the frame by name in SQL.

```python
orders = pl.scan_parquet("orders/*.parquet")
customers = pl.scan_csv("customers.csv")

# frames referenced by Python variable name (locals included)
pl.sql("""
    SELECT c.region, SUM(o.revenue) AS revenue
    FROM orders o JOIN customers c USING (customer_id)
    WHERE o.year = 2024
    GROUP BY c.region
    ORDER BY revenue DESC
""").collect()

# one frame, referenced as `self`
lf.sql("SELECT region, price FROM self WHERE price > 10").collect()

# explicit table names
ctx = pl.SQLContext(orders=orders, customers=customers)
ctx.execute("SELECT * FROM orders LIMIT 5").collect()
```

SQL errors (unknown column, unsupported function) surface at
`collect_schema()` or `collect()`, not when the query is built: run
`.collect_schema()` first.

## Mixing SQL and expressions

A SQL result is an ordinary `LazyFrame`: continue with expressions where
that is simpler, and the whole chain still optimizes as one plan.
`pl.sql_expr("price * qty AS total")` puts a single SQL expression inside
`select()` or `with_columns()`.

## Dialect notes

Polars SQL follows PostgreSQL, with a few differences. CTEs, joins,
subqueries, window functions, `QUALIFY`, and set operations all work, and
DuckDB's generated TPC-H and TPC-DS query sets run on Polars. Things to know:

- `price * 1.1` returns `Decimal`: numeric literals without an exponent are
  typed as Decimal, as in PostgreSQL. Write `1.1::DOUBLE` or `1.1e0` for
  `Float64`.
- Unlike PostgreSQL, `/` on integers returns `Float64`; use `DIV(a, b)` for
  integer division.
- Unlike PostgreSQL, there is no `DATE_TRUNC`: group by `DATE_PART` /
  `EXTRACT` components, or truncate with `.dt.truncate("1mo")` on the
  result.
- On "unsupported function", look the name up in the map below rather than
  guessing an alternative.

## SQL reference map

Fetch `https://docs.pola.rs/api/python/stable/reference/sql/<slug>`:

| Need | Slug |
|---|---|
| Clauses: `SELECT`, `JOIN`, `GROUP BY`, `QUALIFY`, `ORDER BY`, `LIMIT` | `clauses.html` |
| `UNION`, `EXCEPT`, `INTERSECT` | `set_operations.html` |
| `CREATE TABLE`, `EXPLAIN`, `SHOW TABLES`, `UNNEST` | `table_operations.html` |
| Functions: `aggregate`, `array`, `bitwise`, `conditional`, `math`, `string`, `temporal`, `trigonometry`, `types`, `window` | `functions/<category>.html` |
| Operators | `operators/index.html` |
| `pl.sql`, `SQLContext`, `sql_expr` | `python_api.html` |
