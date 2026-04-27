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

## Streaming

```python
import time

# Submit streaming job: Kafka → Iceberg
job_id = (session.stream_source("user-events",
              bootstrap_servers="kafka:9092",
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

### Python UDF

```python
# Register a Python UDF
session.register_udf("double_amount", """
def udf(amount):
    return amount * 2.0
""", return_type="DOUBLE", param_types=["DOUBLE"])

# Use in SQL
result = session.source("SELECT id, double_amount(amount) AS doubled FROM iceberg.db.sales")
```

### Python Map Function

```python
# Apply a map function to each row
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
