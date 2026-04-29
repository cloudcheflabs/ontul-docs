# Python SDK

The Ontul Python SDK connects to Ontul Master via Arrow Flight and provides a simple API for queries, batch processing, and streaming.

## Installation

The Python SDK is included in the Ontul distribution at `lib/python/ontul/`. Install the required dependency and add the SDK to your Python path:

```bash
pip install pyarrow
export PYTHONPATH="lib/python:$PYTHONPATH"
```

## OntulSession

```python
from ontul.session import OntulSession

# Explicit connection
session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# Or via environment variables
# export ONTUL_HOST=localhost
# export ONTUL_PORT=47470
# export ONTUL_USER_TOKEN=your-jwt-token
session = OntulSession()
```

### Connection Options

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| `host` | `ONTUL_HOST` | `localhost` | Ontul Master host |
| `port` | `ONTUL_PORT` | `47470` | Flight SQL port |
| `token` | `ONTUL_USER_TOKEN` | — | JWT Bearer token |
| `access_key` | `ONTUL_USER_ACCESS_KEY` | — | AccessKey (alternative auth) |
| `secret_key` | `ONTUL_USER_SECRET_KEY` | — | SecretKey (alternative auth) |
| `use_tls` | — | `False` | Enable TLS |

## Execute SQL

```python
# Query with results
result = session.source("SELECT * FROM tpch.tiny.customer LIMIT 5")
print(result)

# Get as Pandas DataFrame
df = session.sql_pandas("SELECT c_name, c_acctbal FROM tpch.tiny.customer ORDER BY c_acctbal DESC LIMIT 10")
print(df)

# DDL / DML (no results)
session.execute("CREATE SCHEMA IF NOT EXISTS iceberg.analytics")
session.execute("INSERT INTO iceberg.analytics.events SELECT * FROM tpch.tiny.lineitem LIMIT 100")
```

## DataFrame Operations

```python
# Read from any catalog
df = session.source("SELECT * FROM tpch.tiny.customer")

# Transformations
result = (df
    .filter("c_acctbal > 1000")
    .select("c_custkey", "c_name", "c_acctbal")
    .order_by("c_acctbal", ascending=False)
    .limit(10))

result.show()

# Write to Iceberg
session.source("SELECT * FROM tpch.tiny.customer") \
    .sink("iceberg.warehouse.customers")
```

## Source/Sink by Connection ID

Reference a registered connection (`POST /admin/connections`) by ID instead of repeating endpoints and credentials:

```python
# S3 source/sink by connection ID
(session.source_s3("s3://warehouse/raw/sales", "parquet", connection="warehouse-s3")
    .filter("quantity > 0")
    .with_column("total", "quantity * unit_price")
    .sink_s3("s3://warehouse/curated/sales", "parquet", connection="warehouse-s3"))

# JDBC by connection ID
(session.source_jdbc("analytics-pg", "SELECT * FROM public.orders WHERE dt = CURRENT_DATE")
    .sink("ice.staging.orders"))

session.source("SELECT * FROM ice.warehouse.daily_summary") \
    .sink_jdbc("analytics-pg", "public.daily_summary")
```

See the [Connection ID feature reference](../features/connection-id.md) for the full reference, including overriding stored properties per job.

## Streaming

```python
import time

# Submit streaming job: Kafka → Iceberg, by connection ID
job_id = (session.stream_source("user-events",
              connection="events-kafka",
              format="json",
              auto_offset_reset="earliest")
    .filter("event_type <> 'logout'")
    .select("event_id", "event_type", "user_name", "amount", "event_time")
    .sink("iceberg.analytics.stream_events")
    .commit_interval(5000)
    .start(duration_sec=600))

print(f"Streaming job started: {job_id}")

# Wait and verify
time.sleep(30)
df = session.sql_pandas("SELECT COUNT(*) AS total FROM iceberg.analytics.stream_events")
print(df)
```

## User-Defined Functions (UDFs)

Python UDFs are defined as regular Python functions and registered via `session.register_udf()`. The function is serialized with cloudpickle and shipped to Workers for execution.

### Step 1: Define UDF Functions

```python
def clean_email(email):
    """Trim + lowercase."""
    if email is None:
        return None
    return email.strip().lower()


def email_score(email):
    """Complex scoring logic — any Python code works."""
    if email is None:
        return 0
    e = email.lower()
    score = 0
    if e.endswith('.com'):
        score += 100
    if e.endswith('.io'):
        score += 80
    if 'admin' in e:
        score += 25
    if 'test' in e:
        score -= 10
    return score


def make_greeting(name, greeting):
    """Multi-arg UDF."""
    return f"{greeting}, {name}!"


def is_bot(user_agent):
    """Boolean return type."""
    if user_agent is None:
        return False
    ua = user_agent.lower()
    return 'bot' in ua or 'crawl' in ua or 'spider' in ua
```

### Step 2: Register UDFs

```python
session.register_udf("py_clean_email", clean_email,
                     return_type='string', param_types=['string'])

session.register_udf("py_email_score", email_score,
                     return_type='long', param_types=['string'])

session.register_udf("py_greet", make_greeting,
                     return_type='string', param_types=['string', 'string'])

session.register_udf("py_is_bot", is_bot,
                     return_type='boolean', param_types=['string'])
```

Supported types: `string`, `long`, `int`, `double`, `float`, `boolean`

### Step 3: Use in SQL

```python
# Single-arg UDF
result = session.source(
    "SELECT py_clean_email('  Alice@EXAMPLE.COM  ') AS cleaned "
    "FROM tpch.tiny.nation LIMIT 1")
print(result.to_pandas())
# Output: alice@example.com

# UDF over real columns
result = session.source(
    "SELECT n_name, py_email_score('user@' || LOWER(n_name) || '.com') AS score "
    "FROM tpch.tiny.nation ORDER BY n_nationkey LIMIT 5")
print(result.to_pandas())

# Multi-arg UDF
result = session.source(
    "SELECT n_name, py_greet(n_name, 'Hello') AS msg "
    "FROM tpch.tiny.nation LIMIT 3")
print(result.to_pandas())
# Output: Hello, ALGERIA!

# Boolean UDF
result = session.source(
    "SELECT py_is_bot('Mozilla/5.0 GoogleBot') AS bot1, "
    "       py_is_bot('Mozilla/5.0 Chrome/120') AS bot2 "
    "FROM tpch.tiny.nation LIMIT 1")
print(result.to_pandas())
# Output: bot1=True, bot2=False
```

### Unregister UDF

```python
session.unregister_udf("py_clean_email")

# After unregister, SQL will fail:
try:
    session.source("SELECT py_clean_email('x') FROM tpch.tiny.nation LIMIT 1")
except Exception as e:
    print(f"Expected: {e}")  # function not found
```

### Python Map Function

For row-level transformations without SQL:

```python
def transform_row(row):
    row['amount_usd'] = row['amount'] * 1.1  # EUR to USD
    return row

session.source("SELECT * FROM iceberg.db.sales") \
    .py_map(transform_row) \
    .sink("iceberg.db.sales_usd")
```

## Cross-Catalog Federation

```python
# Join across multiple data sources in a single query
result = session.sql_pandas("""
    SELECT n.name, c.c_name
    FROM neorun.public.orders n
    JOIN tpch.tiny.customer c ON n.customer_id = c.c_custkey
    ORDER BY n.name
""")
print(result)
```

## Complete Example

```python
from ontul.session import OntulSession
import time

session = OntulSession(host="localhost", port=47470, token="your-jwt-token")

# 1. Interactive query
df = session.sql_pandas("SELECT c_name, c_acctbal FROM tpch.tiny.customer ORDER BY c_acctbal DESC LIMIT 5")
print("Top customers:")
print(df)

# 2. Batch ETL: TPC-H → Iceberg
session.execute("CREATE SCHEMA IF NOT EXISTS iceberg.warehouse")
session.execute(
    "CREATE TABLE iceberg.warehouse.top_customers AS "
    "SELECT c_custkey, c_name, c_acctbal FROM tpch.sf1.customer "
    "WHERE c_acctbal > 5000")

# 3. Verify
count = session.sql_pandas("SELECT COUNT(*) AS cnt FROM iceberg.warehouse.top_customers")
print(f"Rows written: {count['cnt'][0]}")

session.close()
```
