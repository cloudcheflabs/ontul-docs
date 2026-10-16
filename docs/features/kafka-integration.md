# Kafka Integration

Ontul supports Kafka (including ItdaStream) as a streaming source, enabling real-time data pipelines with Flink-style continuous processing. Data is consumed event-at-a-time, NOT in micro-batches.

## Streaming Source

Ontul consumes messages from Kafka topics with support for multiple formats:

- **JSON**: Parse JSON messages; the schema is inferred as the union of the fields seen (or declared with `schema`), and a value that does not fit its column is rejected, not coerced
- **Avro**: Deserialize Avro messages using Schema Registry integration
- **Raw**: Access raw key, value, topic, partition, offset, and timestamp fields

Multi-worker consumption: worker `i` of `n` reads partitions `p % n == i` (a fixed assignment, no group rebalancing); new partitions are discovered while the job runs.

## Operations Pipeline

Streaming jobs support a configurable operations pipeline:

```json
{
  "operations": [
    {"type": "FILTER", "value": "amount > 100"},
    {"type": "WINDOW", "value": "TUMBLING(SIZE 10 SECONDS)"},
    {"type": "GROUP_BY", "value": "category"},
    {"type": "AGG", "value": "SUM(amount) as total, COUNT(*) as cnt"}
  ]
}
```

- **FILTER**: Predicate-based row filtering
- **WINDOW**: TUMBLING, SLIDING (with slide interval), SESSION (with gap)
- **GROUP_BY**: Hash-based multi-worker shuffle (STREAM_SHUFFLE opcode)
- **AGG**: SUM, COUNT, AVG, MIN, MAX with alias support

## Streaming Sinks

Processed data can be written to multiple sink types:

| Sink | Without a window | With a window |
|------|------------------|---------------|
| Iceberg `table` (default) | exactly-once | exactly-once |
| JDBC | exactly-once (offsets in the same transaction) | at-least-once (use `mode: upsert` + keys) |
| Kafka `transactional: true` | exactly-once with a Kafka source | at-least-once |
| Kafka (non-tx), NeorunBase, Elasticsearch, HTTP, Lance | at-least-once | at-least-once |

See [Delivery guarantees](../streaming/guide.md#delivery-guarantees) for the conditions.

## Barrier Checkpoint

The master triggers a checkpoint barrier on every worker on a fixed interval. Each worker captures
its source offsets, aligns the barrier across its keyBy inputs, pre-commits or commits its sink and
writes offsets + watermarks + window state to the Exchange Manager, then acknowledges. A checkpoint
is a restore point only when every worker has acknowledged it; offsets are committed to the source
only then — never on a timer, and never when a job fails. Details:
[Checkpointing](../streaming/guide.md#checkpointing).

## Use Cases

### Real-Time Ingestion with Transform

```
Kafka → FILTER + WINDOW + AGG → Iceberg Table
Kafka → FILTER → NeorunBase (REST bulk-insert)
```

### Stream-to-Stream

```
Kafka Input → Transform → Kafka Output (transactional)
```

### CDC Pipeline

```
NeorunBase (Iceberg CDC) → Polaris → Ontul SQL query
```

## Job Management

Streaming jobs are managed through the Admin UI or REST API:

- Submit, kill, and monitor jobs (multiple jobs concurrently)
- View real-time job logs from all Workers (multi-worker log merge)
- Monitor checkpoint progress
- Configure consumer properties (group ID, offset reset, poll timeout)
