# Arrow-Native Execution Engine

Ontul's execution engine is built entirely on Apache Arrow, processing all data in Arrow columnar format from ingestion to output. This eliminates unnecessary serialization/deserialization overhead and enables vectorized computation.

## Arrow Columnar Format

All internal data is represented as Arrow RecordBatches. Data is converted to Arrow format once at the source (connector) and converted out once at the destination (client or sink). Everything in between — filtering, joining, aggregating, sorting — operates directly on Arrow vectors with zero-copy.

## Vectorized Operator Pipeline

The execution engine implements a pull-based streaming operator pipeline:

- **ScanOperator**: Reads data splits from connectors as Arrow RecordBatches
- **FilterOperator**: Evaluates predicates on Arrow vectors
- **ProjectOperator**: Computes expressions and selects columns
- **HashJoinOperator**: Hash-based join with Arrow vector probing
- **SortMergeJoinOperator**: Sort-merge join for large datasets
- **HashAggregateOperator**: Hash-based aggregation on Arrow vectors
- **SortOperator**: Sorts Arrow RecordBatches
- **TopNOperator**: Efficient top-N selection without full sort
- **WindowOperator**: Window functions (ROW_NUMBER, RANK, DENSE_RANK, COUNT, SUM, AVG, MIN, MAX) with PARTITION BY and ORDER BY
- **ExchangeOperator**: Shuffles data between Workers via Arrow Flight

Each operator follows an `open()` → `next()` → `close()` lifecycle, streaming data through the pipeline without materializing full intermediate results.

## Java-Native Implementation

The execution engine is implemented purely in Java using the Arrow Java libraries (`arrow-vector`, `arrow-algorithm`). There are no JNI dependencies, no native engines (Velox, DataFusion), and no external runtime requirements — ensuring portability and simplifying deployment.

## Memory Management & Disk Spill

When operators exceed the memory threshold, intermediate results are spilled to local disk via SpillStore and automatically cleaned up after query completion. Worker local disk is used exclusively for temporary spill — Ontul is a processing engine, not a storage engine.

## Data Transport

Arrow Flight is used for all bulk data transfer between nodes:

- Query results from Workers to Master to Client
- Shuffle data between Workers (Worker-to-Worker)
- Source/Sink data transfers for jobs

All data stays in Arrow IPC binary format throughout — no JSON serialization, no base64 encoding.
