# Data Types

Every SQL type Ontul accepts is a real type all the way through the engine: a `DATE` column is read as a
date, compared as a date, sorted as a date, aggregated as a date, written to Iceberg as a date and shown
to you as `2024-02-29` — never as the number it happens to be stored as. A `DECIMAL(12,2)` is summed
exactly, keeps its scale, and is written back as `decimal(12,2)`.

This matters most where two engines share a table. The tables Ontul writes are read by Trino, Spark and
Flink with the declared types, and the tables those engines write are read by Ontul with the same
values. Ontul's end-to-end suite checks exactly that against Trino 479 on a shared Polaris catalog
(`tests/e2e-datetime-decimal.sh`).

## Type reference

| SQL type | Iceberg type | In the engine (Arrow) | Shown as | Java SDK value |
|---|---|---|---|---|
| `BOOLEAN` | `boolean` | Bool | `true` | `Boolean` |
| `TINYINT` / `SMALLINT` / `INTEGER` | `int` | Int(8/16/32) | `42` | `Byte` / `Short` / `Integer` |
| `BIGINT` | `long` | Int(64) | `42` | `Long` |
| `REAL` / `FLOAT` | `float` | Float(32) | `1.5` | `Float` |
| `DOUBLE` | `double` | Float(64) | `1.5` | `Double` |
| `DECIMAL(p, s)` | `decimal(p, s)` | Decimal(p, s) | `12.30` | `BigDecimal` |
| `VARCHAR` / `CHAR` | `string` | Utf8 | `abc` | `String` |
| `VARBINARY` | `binary` | Binary | — | `byte[]` |
| `DATE` | `date` | Date(DAY) | `2024-02-29` | `LocalDate` |
| `TIME` | `time` | Time(µs) | `10:11:12` | `LocalTime` |
| `TIMESTAMP` | `timestamp` | Timestamp(µs) | `2024-02-29 13:45:10.5` | `LocalDateTime` |
| `TIMESTAMP WITH LOCAL TIME ZONE` | `timestamptz` | Timestamp(µs, UTC) | `2024-02-29 13:45:10 UTC` | `Instant` |
| `ROW(...)` / `ARRAY(...)` / `MAP(...)` | `struct` / `list` / `map` | Struct / List / Map | — | `Map` / `List` |

Notes:

- **Precision.** Timestamps and times carry microseconds, which is Iceberg's precision — in columns,
  in literals (`TIMESTAMP '2024-01-01 10:00:00.123456'`) and in `CAST(x AS TIMESTAMP)`.
- **Time zone.** The engine's session time zone is UTC. `TIMESTAMP` has no zone: it is the wall-clock
  value stored in the table. A `timestamptz` column is an instant, read and shown in UTC.
- **DECIMAL precision** is at most 38 digits. A `DECIMAL` declared without precision or scale is
  `DECIMAL(38, 18)`.
- **Results.** The SQL query page and `POST /admin/query/execute` return integers and floating point as
  JSON numbers. They return a `DECIMAL` as text, so its scale survives JavaScript (`"12.30"`, not
  `12.3`), and datetimes as the text shown above. JDBC clients over Flight SQL receive native
  `java.sql.Date` / `Timestamp` / `BigDecimal` values. The MCP server renders the same text in its
  Markdown tables.

## Literals

```sql
SELECT DATE '2024-02-29',
       TIME '10:11:12',
       TIMESTAMP '2024-02-29 13:45:10.500',
       12.34,                      -- an exact DECIMAL(4,2)
       1.5E0,                      -- a DOUBLE
       INTERVAL '3' DAY,
       INTERVAL '1' MONTH;
```

A numeric literal with a decimal point is an exact `DECIMAL`, not a `DOUBLE`. Write `1.5E0` or
`CAST(1.5 AS DOUBLE)` when you want floating point.

## Comparisons

Values compare by their type:

- Numbers compare by value. Integers and decimals compare exactly; only a floating-point operand makes
  the comparison floating point.
- Datetimes compare in time order. A `DATE` compared with a `TIMESTAMP` is read as midnight of that day.
- Text compared with a datetime is read as that datetime. `d < '2020-01-01'` compares dates.
- Text compares case-sensitively.

```sql
SELECT * FROM ice.sales.orders WHERE order_date >= DATE '2024-01-01' AND order_date < DATE '2024-02-01';
SELECT * FROM ice.sales.orders WHERE created_at > TIMESTAMP '2024-01-31 23:00:00';
SELECT * FROM ice.sales.orders WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31';
SELECT * FROM ice.sales.orders WHERE price = 12.3;   -- matches 12.30
```

## CAST

| From → To | Result |
|---|---|
| `DATE` → `TIMESTAMP` | midnight of the day |
| `TIMESTAMP` → `DATE` | the date part |
| datetime → `VARCHAR` | the text shown above (`2024-02-29`, `2024-02-29 13:45:10`) |
| `VARCHAR` → `DATE` / `TIMESTAMP` / `TIME` | ISO text, with a space or `T` between date and time; a trailing offset (`+09:00`, `Z`) is converted to UTC |
| any number → `DECIMAL(p, s)` | rounded half-up to `s` digits |
| `DECIMAL` / `DOUBLE` → integer | truncated toward zero |

Text that is not a value of the target type casts to `NULL`. For example, `CAST('x' AS DATE)` is `NULL`
and `CAST('12a' AS INTEGER)` is `NULL`.

A value that **is** valid but does not fit is an error, not a `NULL`:

- `CAST(123.45 AS DECIMAL(3,2))` fails the query.
- A `BIGINT` that overflows `INTEGER` fails the query.
- Division by zero fails the query.

## Date and time functions

| Function | Result |
|---|---|
| `EXTRACT(unit FROM x)` | `BIGINT`. Units: `YEAR`, `QUARTER`, `MONTH`, `WEEK` (ISO week), `DAY`, `DOW` (Sunday = 1), `ISODOW` (Monday = 1), `DOY`, `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND`, `MICROSECOND`, `EPOCH` (seconds), `DECADE`, `CENTURY`, `MILLENNIUM`, `ISOYEAR` |
| `YEAR(x)`, `QUARTER(x)`, `MONTH(x)`, `WEEK(x)`, `DAYOFMONTH(x)`, `DAYOFWEEK(x)`, `DAYOFYEAR(x)`, `HOUR(x)`, `MINUTE(x)`, `SECOND(x)` | the same as `EXTRACT` with that unit |
| `FLOOR(x TO unit)`, `CEIL(x TO unit)` | `x` truncated down or up to `YEAR`, `QUARTER`, `MONTH`, `WEEK` (weeks start on Sunday), `DAY`, `HOUR`, `MINUTE`, `SECOND`; keeps `x`'s type |
| `x + INTERVAL '3' DAY`, `x - INTERVAL '2' HOUR` | `x` moved by the interval. Day–time intervals are exact durations; `MONTH` / `YEAR` intervals move the calendar, so `DATE '2020-01-31' + INTERVAL '1' MONTH` is `2020-02-29` |
| `TIMESTAMPADD(unit, n, x)` | `x + n` units |
| `TIMESTAMPDIFF(unit, a, b)` | whole units from `a` to `b` |
| `CURRENT_DATE`, `CURRENT_TIMESTAMP`, `LOCALTIMESTAMP`, `CURRENT_TIME` | now, in UTC |

The `DOW` and `WEEK` definitions are Calcite's. The planner folds constant expressions with the same
rules, so a folded result and an evaluated result agree. Trino's `day_of_week()` counts from Monday = 1,
which is `ISODOW` here.

## DECIMAL arithmetic

`+`, `-`, `*`, `/` and `%` on decimals are exact. Each result is rounded once, half-up, to the scale of the
result type the planner derives:

```sql
SELECT 0.10 + 0.20;                  -- 0.30, not 0.30000000000000004
SELECT price * quantity FROM ...;    -- DECIMAL(p1+p2, s1+s2)
SELECT price / 3 FROM ...;           -- DECIMAL with scale 6, e.g. 4.100000
```

`SUM` over an integer column is an exact `BIGINT`, as in Trino: it never passes through floating point,
so values past 2^53 keep every digit, and a sum that overflows `BIGINT` fails the query. `AVG` over an
integer column is a `DOUBLE` (`AVG` of 1 and 2 is 1.5, not 1).

`SUM` and `AVG` over a `DECIMAL` column accumulate exactly as well, on each worker and in the
coordinator's final combine. `SUM` of a `DECIMAL(p, s)` is a `DECIMAL(38, s)` and keeps the column's scale, and `AVG` rounds half-up to it. `MIN` and
`MAX` return the column's own type.

Arithmetic that mixes a `DECIMAL` with a `DOUBLE` is floating point, as SQL defines it.

## Writing the types

`CREATE TABLE` declares them directly. Other engines then read the table with the same types:

```sql
CREATE TABLE ice.sales.orders (
  id         BIGINT,
  order_date DATE,
  created_at TIMESTAMP,
  paid_at    TIMESTAMP WITH TIME ZONE,     -- also TIMESTAMPTZ
  price      DECIMAL(12, 2),
  cutoff     TIME
);

INSERT INTO ice.sales.orders
VALUES (1, DATE '2024-02-29', TIMESTAMP '2024-02-29 13:45:10', TIMESTAMP '2024-02-29 04:45:10',
        12.34, TIME '10:11:12');

UPDATE ice.sales.orders SET price = 99.99, order_date = DATE '2024-03-01' WHERE id = 1;
```

`CREATE TABLE ... AS SELECT` takes each column's type from the query, so
`CAST(o_orderdate AS TIMESTAMP)` becomes a `timestamp` column and `CAST(x AS DECIMAL(12,2))` becomes a
`decimal(12,2)` column. `ALTER TABLE ... ADD COLUMN` accepts the same type names.

Every write path — `INSERT`, CTAS, `UPDATE`, `MERGE` and streaming sinks — converts a value to the
column's type in one place. A value the column cannot hold fails the write and names the column. It is
never stored as text, and never replaced with `NULL`.

## How connectors map types

| Source | DATE | TIME | TIMESTAMP | DECIMAL |
|---|---|---|---|---|
| Iceberg (Parquet / ORC / Avro data files, any writer) | `date` | `time` | `timestamp`, `timestamptz` | `decimal(p, s)` |
| Parquet files (`file` connector) | `DATE` | `TIME(MILLIS/MICROS/NANOS)` | `TIMESTAMP(MILLIS/MICROS/NANOS)`, legacy `INT96` | `DECIMAL` as INT32, INT64 or bytes |
| Avro files | `date` | `time-millis`, `time-micros` | `timestamp-*` (an instant), `local-timestamp-*` | `decimal` |
| ORC files | `DATE` | — | `TIMESTAMP` | `DECIMAL` |
| JDBC (PostgreSQL, MySQL, ...) | `DATE` | `TIME` | `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE` | `NUMERIC(p, s)` (unconstrained `NUMERIC`: `DECIMAL(38, s)`) |
| Elasticsearch | — | — | `date`, read from epoch millis or ISO text | — |
| TPC-H / TPC-DS | `DATE` columns | — | — | — (prices are `DOUBLE`) |

A Parquet `DECIMAL` can be stored as INT32, INT64 or unscaled bytes, depending on the writer. Trino
writes every decimal as bytes. Ontul reads all three, on both the row-at-a-time reader and the columnar
reader.

Outbound sinks:

- **JDBC sinks** bind `DATE` / `TIME` / `TIMESTAMP` / `DECIMAL` as those JDBC types.
- **JSON sinks** — Elasticsearch, HTTP, NeorunBase REST and Kafka — write datetimes as ISO-8601 text
  (`2024-02-29T13:45:10`), which Elasticsearch maps as a `date`, and decimals as JSON numbers.

## When a value cannot be read

A value a reader cannot decode fails the query and names the column and the file. Before, such a
value became a `NULL` with a debug log line, which is how a Trino-written `DECIMAL` once read as all
`NULL`s without an error. The same rule applies to a failed expression in a projection, and to a batch
that cannot be read — that used to end the file early.

The only `NULL`s a reader produces are real ones:

- a `NULL` stored in the file
- a column the file predates, for example one added by `ALTER TABLE ADD COLUMN` after the file was written
