# String, Temporal, List, Struct Expressions

Reference for the `str`, `dt`, `list`, and `struct` namespaces, `pl.selectors`,
casting, and `when/then/otherwise`. Load this file when a task involves text
processing, date/time arithmetic, nested data, or multi-type column selection.

## Contents

- String — `str.*` (includes parsing strings to dates/datetimes)
- Temporal — `dt.*` (includes window grouping on timestamps)
- List — `list.*`
- Struct — `struct.*`
- Selectors — `polars.selectors`
- Casting
- `when / then / otherwise`

---

## String — `str.*`

```python
pl.col("text").str.contains("pattern")          # regex by default
pl.col("text").str.contains("prefix", literal=True)  # literal match
pl.col("text").str.starts_with("pre")
pl.col("text").str.ends_with("suf")
pl.col("text").str.len_chars()                  # character count (not bytes)
pl.col("text").str.len_bytes()                  # byte count

pl.col("text").str.to_lowercase()
pl.col("text").str.to_uppercase()
pl.col("text").str.strip_chars()                # strip whitespace both sides
pl.col("text").str.strip_chars_start()
pl.col("text").str.strip_chars_end()
pl.col("text").str.strip_chars(" ,-")           # strip specific characters

pl.col("text").str.replace("old", "new")        # first match, literal
pl.col("text").str.replace("pat", "new", n=1)   # n-th match
pl.col("text").str.replace_all("old", "new")    # all matches
pl.col("text").str.replace_all(r"\s+", " ")     # regex

pl.col("text").str.slice(0, 5)                  # chars 0..5
pl.col("text").str.head(3)                      # first 3 chars
pl.col("text").str.tail(3)                      # last 3 chars

pl.col("text").str.split(",")                   # List[Str]
pl.col("text").str.split_exact(",", n=2)        # fixed-width struct
pl.col("text").str.splitn(",", n=3)             # at most n parts

pl.col("text").str.extract(r"(\d+)", group_index=1)   # first capture group
pl.col("text").str.extract_all(r"\d+")                # List[Str] of all matches

pl.col("text").str.zfill(5)                     # zero-pad to width 5
pl.col("text").str.pad_start(8, "0")            # left-pad to width 8
pl.col("text").str.pad_end(8, " ")              # right-pad to width 8

pl.col("text").str.to_integer(base=10, strict=False)  # parse to Int64
pl.col("text").str.to_decimal(scale=2)                # parse to Decimal (scale = decimal places)
```

### Parse strings to dates and datetimes

```python
# ISO strings like "2024-01-15"
pl.col("d").str.to_date()

# ISO datetimes like "2024-01-15T09:30:00"
pl.col("d").str.to_datetime()

# Custom format — pandas directives carry over, except %f (microseconds)
# becomes %.f in Polars (%.f parses ".123456" correctly)
pl.col("d").str.to_datetime("%Y-%m-%d %H:%M:%S")
pl.col("d").str.to_datetime("%Y-%m-%d %.f")

# When strings contain time but you want date precision: parse then truncate
pl.col("d").str.to_datetime().dt.truncate("1d")

# Non-strict parse: return null on failure instead of raising
pl.col("d").str.to_date(strict=False)
pl.col("d").str.to_datetime(strict=False)
```

---

## Temporal — `dt.*`

```python
# Extract components
pl.col("ts").dt.year()
pl.col("ts").dt.month()         # 1-12
pl.col("ts").dt.day()           # 1-31
pl.col("ts").dt.hour()
pl.col("ts").dt.minute()
pl.col("ts").dt.second()
pl.col("ts").dt.microsecond()   # 0-999999
pl.col("ts").dt.weekday()       # 1=Monday, 7=Sunday
pl.col("ts").dt.week()          # ISO week number
pl.col("ts").dt.ordinal_day()   # day of year (1-366)
pl.col("ts").dt.quarter()       # 1-4

# Round and truncate
pl.col("ts").dt.truncate("1mo")   # floor to month start
pl.col("ts").dt.truncate("1d")    # floor to day start
pl.col("ts").dt.truncate("1h")    # floor to hour
pl.col("ts").dt.truncate("15m")   # floor to 15-minute bucket
pl.col("ts").dt.round("1h")       # round to nearest hour

# Shift by duration
pl.col("ts").dt.offset_by("1mo")    # add 1 calendar month
pl.col("ts").dt.offset_by("-7d")    # subtract 7 days
pl.col("ts").dt.offset_by("2h30m")  # add 2h 30min

# Format as string
pl.col("ts").dt.strftime("%Y-%m")     # e.g. "2024-03"
pl.col("ts").dt.strftime("%Y-%m-%d")  # e.g. "2024-03-15"

# Time zone
pl.col("ts").dt.replace_time_zone("UTC")               # attach tz (naive -> aware)
pl.col("ts").dt.convert_time_zone("Europe/Amsterdam")  # convert between tz

# Type conversions
pl.col("ts").dt.date()      # Datetime -> Date
pl.col("ts").dt.time()      # Datetime -> Time
pl.col("ts").dt.epoch(time_unit="s")   # seconds since Unix epoch

# Duration arithmetic
pl.col("ts") + pl.duration(days=7)
pl.col("end") - pl.col("start")           # Duration column
(pl.col("end") - pl.col("start")).dt.total_seconds()  # integer seconds
(pl.col("end") - pl.col("start")).dt.total_minutes()  # integer minutes
(pl.col("end") - pl.col("start")).dt.total_hours()    # integer hours
```

### Window grouping on timestamps

```python
# Group events into 5-minute buckets
lf.with_columns(
    bucket=pl.col("ts").dt.truncate("5m")
).group_by("bucket").agg(pl.len().alias("events"))

# Window expression using truncated ts as the over() key
lf.with_columns(
    bucket_total=pl.col("amount").sum().over(pl.col("ts").dt.truncate("1d"))
)
```

---

## List — `list.*`

List columns hold a variable-length list per row. Use `explode()` to turn
them into individual rows.

```python
pl.col("tags").list.len()                   # length of each list
pl.col("tags").list.first()                 # first element
pl.col("tags").list.last()                  # last element
pl.col("tags").list.get(2)                  # element at index 2
pl.col("tags").list.slice(1, 3)             # sublist [1, 2, 3]
pl.col("tags").list.head(2)                 # first 2 elements
pl.col("tags").list.tail(2)                 # last 2 elements

pl.col("nums").list.sum()
pl.col("nums").list.mean()
pl.col("nums").list.min()
pl.col("nums").list.max()

pl.col("tags").list.contains("python")      # Boolean, one per row
pl.col("tags").list.sort()
pl.col("tags").list.unique()                # order not preserved
pl.col("tags").list.sort(descending=True)
pl.col("nums").list.reverse()

pl.col("a").list.concat(pl.col("b"))        # concatenate two list columns
pl.col("tags").list.join(", ")              # join into a single string

# Explode: one row per list element
lf.explode("tags")

# Apply expression to each element without leaving the engine
pl.col("scores").list.eval(pl.element() * 2)
pl.col("scores").list.eval(pl.element().filter(pl.element() > 5))
```

---

## Struct — `struct.*`

Struct columns hold named fields per row, similar to a JSON object column.

```python
# Access a field
pl.col("address").struct.field("city")
pl.col("address").struct.field("zip_code")

# Rename fields
pl.col("address").struct.rename_fields(["town", "postcode"])

# Unnest: promote all struct fields to top-level columns
lf.unnest("address")

# Build a struct from multiple columns
pl.struct("city", "country").alias("location")
pl.struct(pl.col("lat"), pl.col("lon")).alias("coords")

# Modify a field inside the struct
pl.col("address").struct.with_fields(
    pl.field("city").str.to_uppercase()
)

# Parse JSON-embedded columns
pl.col("payload").str.json_decode()    # String -> Struct or List
```

---

## Selectors — `polars.selectors`

Selectors let you refer to groups of columns by type or name pattern without
listing column names explicitly.

```python
import polars.selectors as cs

# By dtype family
cs.numeric()        # Int*, UInt*, Float*
cs.integer()        # Int*, UInt*
cs.float()          # Float32, Float64
cs.string()         # Utf8 / String
cs.boolean()
cs.temporal()       # Date, Datetime, Duration, Time
cs.categorical()
cs.binary()         # raw bytes columns

# By dtype instance
cs.by_dtype(pl.Int64, pl.Float64)

# By name pattern
cs.starts_with("sales_")
cs.ends_with("_id")
cs.contains("revenue")
cs.matches(r"^q\d$")    # regex; anchors are optional

# Set operations
cs.numeric() | cs.temporal()               # union
cs.numeric() - cs.ends_with("_id")         # difference
~cs.numeric()                              # complement (all non-numeric)
cs.numeric() & cs.starts_with("net_")     # intersection

# Use anywhere a column selector is accepted
lf.select(cs.numeric())
lf.select((cs.float() * 1.21).name.suffix("_with_tax"))
lf.with_columns(cs.string().str.to_uppercase())
lf.drop(cs.temporal())
lf.select(pl.all().exclude(cs.categorical()))
```

---

## Casting

```python
# Strict (default) — raises on values that cannot be cast
pl.col("price").cast(pl.Float64)
pl.col("qty").cast(pl.Int32)
pl.col("flag").cast(pl.Boolean)
pl.col("code").cast(pl.Categorical)

# Non-strict — converts unparseable values to null instead of raising
pl.col("price").cast(pl.Float64, strict=False)
pl.col("qty").cast(pl.Int32, strict=False)

# Common patterns
pl.col("ts_ms").cast(pl.Datetime("ms"))    # epoch ms to Datetime
pl.col("date_int").cast(pl.Date)           # epoch days to Date

# Inspect the dtype before casting
lf.collect_schema()["col_name"]
```

---

## when / then / otherwise

Strings inside `then()` and `otherwise()` are **column names**, not values.
Wrap literal values in `pl.lit()`.

```python
# Basic conditional
pl.when(pl.col("age") >= 18)
  .then(pl.lit("adult"))
  .otherwise(pl.lit("minor"))
  .alias("age_group")

# Multiple conditions (CASE WHEN)
pl.when(pl.col("score") >= 90).then(pl.lit("A"))
  .when(pl.col("score") >= 80).then(pl.lit("B"))
  .when(pl.col("score") >= 70).then(pl.lit("C"))
  .otherwise(pl.lit("F"))
  .alias("grade")

# Value from another column
pl.when(pl.col("is_prime"))
  .then(pl.col("revenue"))
  .otherwise(pl.lit(0))
  .alias("prime_revenue")

# Null handling
pl.col("v").fill_null(0)
pl.col("v").fill_null(strategy="forward")
pl.col("v").fill_null(strategy="backward")

# SUM(CASE WHEN ...) inside agg()
lf.group_by("dept").agg(
    pl.when(pl.col("status") == "active")
      .then(pl.col("salary"))
      .otherwise(pl.lit(0))
      .sum()
      .alias("active_payroll")
)
```
