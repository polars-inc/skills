# SQL Interface

Polars runs SQL on the same lazy engine as expressions: a query compiles to
the same optimized plan, so it is as fast as the equivalent expression chain,
streams and spills the same way, and runs on Polars Nanook unchanged. Use SQL
when the user writes or asks for SQL, or when porting SQL queries; otherwise
write expressions.

## Contents

Entry points; parameters; mixing SQL and expressions; writing results;
dialect notes; newer SQL features; querying databases; SQL reference map.

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

`DataFrame.sql()` returns a `DataFrame`; the other entry points return a
`LazyFrame`. Syntax errors raise when the query is built; unknown columns
and unsupported functions surface at `collect_schema()` or `collect()`, so
run `.collect_schema()` before executing.

## Parameters (Polars >= 2.2)

Pass user-supplied values as parameters, never by formatting them into the
query string. A parameter is bound as a literal, so it cannot inject SQL:

```python
pl.sql("SELECT * FROM orders WHERE revenue > $1 AND region = $2",
       params=[100, "EU"])
pl.sql("SELECT * FROM orders WHERE region = :region", params={"region": "EU"})
```

`?` and `$1` take a list; `:name` and `$name` take a dict. `lf.sql()` and
`SQLContext.execute()` accept `params=` too.

## Mixing SQL and expressions

A SQL result is an ordinary `LazyFrame`: continue with expressions where
that is simpler, and the whole chain still optimizes as one plan.
`pl.sql_expr("price * qty AS total")` puts a single SQL expression inside
`select()` or `with_columns()`.

## Writing results and large data

SQL results stream and spill like any lazy query
(`references/memory.md`). For results larger than memory, sink instead of
collecting:

```python
pl.sql("SELECT ... FROM orders").sink_parquet("out.parquet")
```

On Polars >= 2.2, SQL can write files itself with `COPY`. The statement is
lazy: nothing is written until `.collect()`, which returns an empty frame.

```python
ctx.execute("COPY (SELECT * FROM orders WHERE year = 2024) TO 'y2024.parquet'").collect()
ctx.execute("COPY orders FROM 'more_orders.csv' (HEADER true)").collect()
```

The format comes from `(FORMAT parquet|csv|ipc|ndjson)` or the file
extension. `SQLContext(allow_file_reads=False, allow_file_writes=False)`
blocks file access when the SQL comes from untrusted users.

`CREATE TABLE t AS SELECT ...`, `INSERT`, `UPDATE`, `DELETE` and
`CREATE VIEW` change only the tables registered in the `SQLContext`. They
never change the source frames or files.

## Dialect notes

Polars SQL follows PostgreSQL, with a few differences. CTEs, joins,
subqueries, window functions, `QUALIFY`, and set operations all work, and
DuckDB's generated TPC-H and TPC-DS query sets run on Polars. Things to know:

- Numeric literals without an exponent are typed as Decimal, as in
  PostgreSQL: an integer or Decimal column times `1.1` returns `Decimal`
  (a Float64 column times `1.1` stays Float64). Write `1.1::DOUBLE` or
  `1.1e0` for `Float64`.
- Unlike PostgreSQL, `/` on integers returns `Float64`; use `DIV(a, b)` for
  integer division.
- `ASOF JOIN` takes `MATCH_CONDITION (t.ts >= q.ts)`, not `ON`.
- Select aliases can be reused later in the same `SELECT`:
  `SELECT v * 2 AS d, d + 1 AS e`.
- On "unsupported function", look the name up in the map below or run
  `SHOW FUNCTIONS LIKE 'date%'` rather than guessing an alternative.

## Newer SQL features

Check `pl.__version__` before using these; each raises on older versions.

- **2.1:** `DATE_TRUNC('month', ts)`, `WITH RECURSIVE`, `ASOF JOIN`,
  `GENERATE_SERIES` / `RANGE` as table functions.
- **2.2:** `params=`; `COPY`, `INSERT`, `UPDATE`, `DELETE`, `CREATE VIEW`,
  `DESCRIBE`; `CREATE MACRO`; `PIVOT` / `UNPIVOT`; null-safe join keys with
  `ON a.id IS NOT DISTINCT FROM b.id`; `INTERVAL` steps in
  `GENERATE_SERIES`.

```sql
-- date spine for gap-filling (2.2: INTERVAL step)
SELECT * FROM GENERATE_SERIES(DATE '2026-01-01', DATE '2026-03-31', INTERVAL '1 day') AS cal(d)

-- wide table, one column per region (2.2); every column not named in
-- PIVOT becomes a group key, so select only the ones you need first
SELECT * FROM (SELECT month, region, revenue FROM sales)
PIVOT (SUM(revenue) FOR region IN ('EU', 'US'))
```

## Querying databases

`pl.scan_database` (Polars >= 2.2, unstable) returns a `LazyFrame` over a
table or query in an external database. Selected columns, filters and a
`head()` directly on the scan are pushed into the SQL sent to the database;
`explain()` shows that SQL.

```python
lf = pl.scan_database("orders", "postgresql://user@host/db")   # table name or query
(
    lf.filter(pl.col("status") == "paid")
    .group_by("region")
    .agg(pl.col("amount").sum())
    .collect()
)
```

- Install with `pip install 'polars[database]'`. The extra adds the
  `polars-database` plugin and the ADBC, ConnectorX and SQLAlchemy backends.
- `connection` accepts a URI (`scheme+driver://` picks the driver), a
  SQLAlchemy engine, a live DB-API connection, or a zero-argument factory.
- Bind values with `params=`.
- `pl.io.database.show_drivers()` lists the supported dialects.
- On Polars < 2.2, use eager `pl.read_database(query, connection)` or
  `pl.read_database_uri(query, uri)`. Push the filters and columns into
  the query yourself, then call `.lazy()` on the result.

Docs: `https://docs.pola.rs/api/python/stable/reference/api/polars.scan_database.html`.

## SQL reference map

Fetch `https://docs.pola.rs/api/python/stable/reference/sql/<slug>`:

| Need | Slug |
|---|---|
| Clauses: `SELECT`, `JOIN`, `GROUP BY`, `QUALIFY`, `ORDER BY`, `LIMIT` | `clauses.html` |
| `UNION`, `EXCEPT`, `INTERSECT` | `set_operations.html` |
| `CREATE TABLE`/`VIEW`/`MACRO`, `COPY`, `INSERT`, `UPDATE`, `EXPLAIN`, `SHOW TABLES`, `UNNEST`, `GENERATE_SERIES` | `table_operations.html` |
| Functions: `aggregate`, `array`, `bitwise`, `conditional`, `math`, `string`, `temporal`, `trigonometry`, `types`, `window` | `functions/<category>.html` |
| Operators | `operators/index.html` |
| `pl.sql`, `SQLContext`, `sql_expr` | `python_api.html` |
