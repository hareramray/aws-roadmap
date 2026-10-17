# 14. Data engineering, streaming, analytics, and governance

[Roadmap index](README.md)

**Prerequisites:** [SQL and programming](01-prerequisites.md), [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md), [S3/storage](07-storage-and-backup.md), [databases](08-databases-and-caching.md), [security](09-security-and-compliance.md), and [event integration](10-serverless-and-integration.md).

**Outcome:** Build a trustworthy ingestion-to-analysis pipeline, choose an appropriate processing engine and data layout, enforce access controls, and prove correctness after retries, late data, and schema changes.

## 1. Begin with data requirements

Define the business question before choosing services. Record source ownership, data volume, growth, arrival pattern, freshness requirement, retention, sensitivity, consumers, query patterns, and acceptable recovery time. A daily financial report and a real-time fraud signal have different latency, correctness, and operating requirements.

Distinguish operational systems from analytical systems. OLTP systems optimize individual transactions and current state; OLAP systems optimize scans, aggregations, and historical analysis. Copying production tables into a lake does not automatically produce consistent or meaningful metrics. Define what each metric means, its grain, business time zone, treatment of cancellations, and reconciliation source.

Learn batch, microbatch, streaming, ETL, ELT, full refresh, incremental load, change data capture (CDC), and snapshot-plus-change capture. Understand append-only events versus mutable records. Every ingestion design needs a policy for inserts, updates, deletes, duplicate records, and events arriving after their expected time.

## 2. Map the main service responsibilities

| Requirement | Main options | Important distinction |
| --- | --- | --- |
| Durable analytical storage | S3 general-purpose buckets; S3 Tables for supported table workloads | Object storage, file format, table format, and catalog are different layers |
| Metadata catalog | AWS Glue Data Catalog | Describes datasets; does not by itself validate their quality or authorize every access path |
| Data access governance | Lake Formation, IAM, KMS, resource policies | Governance requires aligned permissions across engines and underlying storage |
| Distributed transformations | AWS Glue, EMR on EC2/EKS, EMR Serverless | Select based on engine/control needs, workload duration, compatibility, and cost |
| Interactive SQL on a lake | Athena | Data layout and scanned data strongly influence performance and cost |
| Analytical warehouse | Redshift provisioned or Serverless | Warehouse modeling, workload management, and query performance still matter |
| Durable event stream | Kinesis Data Streams or MSK | Retention, partitioning, ordering, consumer offsets, and throughput planning |
| Managed stream delivery | Amazon Data Firehose | Buffered delivery/transformation differs from a general-purpose retained stream |
| Stateful stream computation | Amazon Managed Service for Apache Flink | Event time, windows, checkpoints, state, and sink semantics |
| Pipeline orchestration | Step Functions, Glue workflows, MWAA/Airflow | Scheduling and dependency management are separate from executing transformation code |
| Search and interactive log exploration | OpenSearch Service | An index has different modeling and retention needs from a warehouse |
| Business intelligence | Amazon Quick / QuickSight capabilities and compatible BI clients | Governed metric definitions, refresh, sharing, and row-level access |

Amazon Data Firehose is the current name used in its [developer guide](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html). The older Kinesis Data Analytics for SQL Applications service was discontinued; use current Flink guidance for new streaming exercises. See the [SQL application discontinuation notice](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/discontinuation.html). Current [Amazon Quick documentation](https://docs.aws.amazon.com/quicksuite/latest/userguide/what-is.html) describes the product's evolution from QuickSight; older courses may use earlier names.

## 3. Build a lake with understandable layers

Separate raw ingestion, validated/conformed data, and consumer-ready datasets. Names such as bronze/silver/gold are optional; the ownership and guarantees are what matter. Raw data should preserve the information needed for replay, subject to retention and deletion requirements. Curated data should have explicit schemas, quality rules, lineage, and supported consumers.

Use consistent keys and metadata: source system, ingestion time, event time, schema version, source record key, and batch or transaction identifier. Decide whether a file is authoritative, partially written, or ready for consumption. For multipart or multi-file publication, use a manifest or transactional table commit rather than expecting consumers to guess when a dataset is complete.

Learn file formats. CSV is convenient for exchange but weak in types and escaping; JSON supports nested records but can be expensive to scan; Avro provides row-oriented serialization with schemas; Parquet and ORC are columnar formats suited to many analytical scans. Compression, column pruning, and predicate pushdown affect actual bytes read and computation.

Partition by useful, bounded-cardinality filters such as date when the query pattern justifies it. Too many tiny partitions or tiny files increase metadata and scheduling overhead. A partition per customer may be wasteful if there are many small customers. Choose target file sizes through representative measurement and compact small files when needed. Never let raw S3 lifecycle rules remove files still referenced by an active analytical table.

## 4. Understand open table formats and catalogs

Apache Iceberg, Apache Hudi, and Delta Lake add table-level metadata and transaction semantics over files. They can support capabilities such as snapshots, schema evolution, updates, deletes, and incremental consumption, with different implementations and engine support. A Parquet file is not itself an Iceberg table.

Learn snapshot commits, optimistic concurrency, partition evolution, schema field identity, delete representations, time travel, compaction, snapshot expiration, and orphan-file cleanup. Maintenance changes cost and recoverability. Expiring snapshots can make historical queries or rollback impossible; deleting files without consulting table metadata can corrupt the table.

Select a format by the writer and reader engines you actually need. Do not assume a service that reads a format can write it or supports every version/operation. Lake Formation's supported permissions and write-operation restrictions also vary by engine and table format. Check [Athena/Lake Formation table-format support](https://docs.aws.amazon.com/lake-formation/latest/dg/athena-lf.html).

S3 Tables uses dedicated table buckets for Apache Iceberg tables and provides managed maintenance capabilities. Compare this with managing Iceberg files and maintenance in general-purpose S3 buckets. Understand catalog integration, access controls, supported engines, regional availability, and maintenance charges. See [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html) and [table maintenance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-maintenance.html).

## 5. Learn batch processing and query optimization

In Glue, distinguish the Data Catalog, crawlers, jobs, connections, schemas, job bookmarks, workflows, and data-quality features. A crawler infers metadata; it cannot decide whether a source's changed type is a valid business change. Use explicit contracts for important datasets and control how catalog changes are applied.

Study Spark execution: lazy transformations, actions, partitions, shuffles, joins, skew, caching, and serialization. A large shuffle or a hot join key can dominate a job despite abundant workers. Use partition pruning and projection, avoid collecting huge datasets to the driver, and inspect execution plans. Glue job bookmarks help track supported source progress but are not a universal exactly-once output guarantee. Confirm support for the chosen connector and API; some DataFrame paths differ from DynamicFrame features. See [Glue connection behavior](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-connect.html).

For Athena, learn databases/tables, workgroups, query result locations, encryption, access control, scan limits, partition management/projection, CTAS, engine versions, and supported table operations. Secure the result bucket because query output may contain sensitive source data. Avoid `SELECT *` on a large lake merely to inspect a few fields; use appropriate filters and understand that a row limit does not necessarily eliminate scanning.

For Redshift, learn dimensional modeling, fact/dimension grain, distribution and sort design, column encoding, statistics, materialized views, workload management, concurrency, ingestion, and external-table access. Compare Serverless and provisioned operation using workload shape and governance requirements. A serverless capacity model does not remove query tuning, idle-related considerations, spending controls, or access-management work.

## 6. Design streaming for correctness

For Kinesis or Kafka/MSK, learn partition keys, ordering boundaries, producer acknowledgment/retries, retention, consumer groups or equivalent consumer state, checkpointing, replay, and consumer lag. A hot key can overload one partition even when aggregate capacity appears sufficient. Scaling partitions may change ordering or state-management assumptions; plan it deliberately.

For Flink, distinguish event time from processing time. Learn watermarks, tumbling/sliding/session windows, late-event handling, state backends, checkpoints, savepoints, restart behavior, and compatible deployment upgrades. A stream processor's exactly-once state guarantee only becomes an end-to-end guarantee when sources, sinks, and side effects participate appropriately.

For Firehose, understand buffering, transformation, destination behavior, delivery errors, backup/error prefixes, and supported format conversion. It is well suited to managed delivery, but a buffered delivery path may not meet a very tight latency target. Verify destination-specific retry and duplication behavior rather than assigning one universal guarantee to every destination.

Use CDC carefully. Capture a consistent starting snapshot, preserve ordering where needed, record source positions, and handle schema evolution and deletes. An old update arriving after a newer one must not overwrite the correct value. Use source sequence/version rules, merge conditions, and reconciliation.

## 7. Govern data as an operational product

Assign dataset owners, access approvers, freshness targets, quality expectations, and deprecation policies. Maintain descriptions, lineage, classification, allowed uses, and consumer contacts. A catalog full of unnamed columns is searchable storage, not a useful data product.

Align IAM, S3 policies, KMS, Glue catalog access, and Lake Formation grants. Learn database/table permissions, data-location permissions, LF-tags, cross-account sharing, and row/column filtering where supported. A user with direct broad S3 access may bypass an intended query-layer restriction, so inspect every access path. Test both positive and negative access cases.

Define retention and deletion across raw files, curated tables, snapshots, replicas, query results, caches, exports, and backups. Mask or tokenize sensitive fields where the business use permits. Apply data minimization before copying production data into development. Track lineage so a correction can be propagated to dependent aggregates and dashboards.

SageMaker Unified Studio and its catalog/lakehouse capabilities offer an integrated data-and-AI experience. Learn the underlying storage, catalog, permission, and engine concepts first; a unified interface does not remove those responsibilities. See [current SageMaker overview](https://docs.aws.amazon.com/next-generation-sagemaker/latest/userguide/what-is-sagemaker.html).

## 8. Lab A: governed sales lake

Create synthetic sales records with `order_id`, `customer_region`, `event_time`, `status`, `amount`, and `source_version`. Include known duplicates, a corrected order, a late arrival, a malformed record, and a synthetic restricted field.

1. Write the contract: one row per current order in the curated table, valid amount/status rules, UTC storage with explicit business-date conversion, and the source-version conflict rule.
2. Create private raw, curated, rejected, and query-result locations; use dedicated prefixes or buckets with deliberate policies and retention.
3. Upload a small fixed dataset with a manifest and batch identifier. Register metadata deliberately or use a constrained crawler.
4. Transform using a small local job first, then optionally Glue. Quarantine invalid records, deduplicate by business key/version, and produce Parquet or an Iceberg table.
5. Query with Athena in a workgroup with appropriate scan controls. Produce daily accepted-order count and revenue, documenting treatment of refunds/cancellations.
6. Add a second batch containing corrections and late data. Reprocess the first batch and show that final metrics do not double.
7. Apply Lake Formation or another explicitly documented access model. Give one role access to the allowed columns/rows and another role a different view; prevent unintended direct-storage bypass.
8. Create a small dashboard or export with visible freshness and metric definitions. Compare a filtered columnar query with a raw-file query using actual execution statistics.

**Verify:** Input records reconcile into accepted, rejected, superseded, and duplicate categories. Curated keys are unique. Metrics match manually calculated expected values. A late correction changes only the intended result. Unauthorized queries fail. A second identical pipeline run yields the same business output.

**Evidence:** Include the data contract, schema, transformation, reconciliation queries, query statistics, permission tests, lineage sketch, and recovery instructions.

**Cost:** Estimate S3 storage/requests, query scan volume, Glue/EMR execution, catalog and maintenance features, logging, and optional dashboard subscriptions. A tiny learning dataset generally does not justify a continuously running warehouse or cluster. Check the current pricing of every optional extension.

**Cleanup:** Disable schedules and ingestion, stop jobs and streaming applications, delete only lab-owned catalog objects and grants, remove query results and all relevant data versions/snapshots, and delete dedicated compute/workgroups where appropriate. Dropping a table may leave its objects behind; deleting a dataset in a BI tool may leave the source billable.

## 9. Lab B: streaming replay exercise

Produce a bounded sequence of synthetic events into Kinesis Data Streams or a suitable Kafka test environment. Aggregate counts by event-time window using an appropriate consumer or Flink. Include out-of-order and duplicate events, stop a consumer, allow backlog to build briefly, then recover from its recorded position.

**Verify:** Explain the ordering boundary, maximum acceptable lag, late-data behavior, and final count. Replay a known range into a separate output and reconcile it with the original. Demonstrate what happens when a sink acknowledges a write but the consumer loses the acknowledgment. Keep the input rate and runtime bounded.

**Cleanup:** Stop producers first, then consumers/applications, and remove dedicated streams, brokers, checkpoints, output objects, and logs. Managed streaming compute can continue charging while no business events arrive.

## Mastery checklist

- [ ] I can distinguish file format, table format, catalog, and query engine.
- [ ] I can choose batch, CDC, or streaming from freshness/correctness requirements.
- [ ] I can explain duplicate handling, late data, delete propagation, and replay.
- [ ] I can diagnose small files, hot partitions, skew, or excessive query scans.
- [ ] I can enforce and test data permissions across all access paths.
- [ ] I can reconcile source records to business metrics after failure and reprocessing.
- [ ] I have a maintenance, retention, cost, and cleanup plan for every data layer.

## Official references

- [AWS Glue developer guide](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html).
- [Athena user guide](https://docs.aws.amazon.com/athena/latest/ug/what-is.html).
- [Redshift management guide](https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html).
- [EMR documentation](https://docs.aws.amazon.com/emr/).
- [Lake Formation guide](https://docs.aws.amazon.com/lake-formation/latest/dg/what-is-lake-formation.html).
- [Managed Service for Apache Flink](https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html).
- [Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html) and [Amazon MSK](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html).
- [Apache Iceberg documentation](https://iceberg.apache.org/docs/latest/).

**Review date:** 2026-10-18. Verify engine versions, connector compatibility, format-specific permissions, service lifecycle, and pricing before building.
