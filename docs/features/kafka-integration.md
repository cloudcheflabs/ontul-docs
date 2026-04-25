# Kafka Integration

Ontul supports Kafka (including ItdaStream) as a streaming source, enabling real-time data pipelines with Flink-style continuous processing. Data is consumed event-at-a-time, NOT in micro-batches.

## Streaming Source

Ontul consumes messages from Kafka topics with support for multiple formats:

- **JSON**: Parse JSON messages with automatic schema inference from the first message
- **Avro**: Deserialize Avro messages using Schema Registry integration
- **Raw**: Access raw key, value, topic, partition, offset, and timestamp fields

Multi-worker consumption: Workers share a consumer group and partitions are automatically distributed across Workers.

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

| Sink | Mode | Exactly-Once |
|------|------|:---:|
| Iceberg | Parquet → AppendFiles commit | O |
| NeorunBase | REST bulk-insert (high throughput) | at-least-once |
| NeorunBase | JDBC batch insert | O |
| Kafka | Producer (transactional mode available) | O (with tx) |
| JDBC | Batch insert | O |
| Elasticsearch | Bulk API | at-least-once |
| HTTP | JSON POST | at-least-once |
| Console | Job log output (debugging) | — |

## Barrier Checkpoint

Ontul uses Master-coordinated barrier checkpoint for distributed streaming:

1. Master triggers `CHECKPOINT_TRIGGER` to all Workers (periodic, configurable interval)
2. Each Worker: flush sink → commit transaction → snapshot state to Exchange Manager
3. Commit Kafka consumer offsets (AFTER sink commit)
4. Report `CHECKPOINT_COMPLETE` back to Master
5. Master: all Workers acked → checkpoint globally complete

This ensures exactly-once for transactional sinks — if a Worker crashes, it restores from the last checkpoint and replays from committed Kafka offsets.

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
