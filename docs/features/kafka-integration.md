# Kafka Integration

Ontul supports Apache Kafka as both a streaming source and sink, enabling real-time data pipelines within the same cluster used for batch processing and interactive SQL.

## Streaming Source

Ontul consumes messages from Kafka topics with support for multiple formats:

- **JSON**: Parse JSON messages with automatic schema inference or a provided schema
- **Avro**: Deserialize Avro messages using Schema Registry integration
- **Raw**: Access raw key, value, topic, partition, offset, and timestamp fields

Each consumer group runs as an independent long-lived streaming job on a Worker node.

## Streaming Sink

Processed data can be written back to Kafka topics:

- Arrow rows are serialized to JSON and produced to the target topic
- Optional key field for partitioning
- Batched writes with LZ4 compression
- Kafka producer properties are fully configurable

## Use Cases

### Real-Time Ingestion

Consume events from Kafka and write directly into Iceberg tables for near-real-time analytics:

```
Kafka Topic → Ontul Streaming Job → Iceberg Table
```

### Stream Processing

Read from one Kafka topic, transform data with SQL or SDK operations, and write to another:

```
Kafka Input Topic → Ontul Processing → Kafka Output Topic
```

## Job Management

Kafka streaming jobs are managed through the Admin UI or REST API:

- Start and stop consumer groups
- Monitor ingestion progress and throughput
- View real-time job logs
- Configure consumer properties (group ID, offset reset, poll timeout, batch size)
