# Data engineering and analytics tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../14-data-engineering-and-analytics.md)

Start with [a governed batch lake](01-governed-batch-lake.md), then [streaming and warehouse design](02-streaming-and-warehouse-design.md). Both provide small deterministic local datasets and correctness checks before optional AWS work. Distributed services are introduced when the workload needs them, with explicit deployment, observation, and cleanup boundaries.

| Roadmap topics | Worked coverage |
| --- | --- |
| Business grain, OLTP/OLAP, batch/microbatch/streaming, ETL/ELT, incremental loads, quality and reconciliation | Tutorial 1 reconciles seven source rows into five accepted orders with one duplicate and one invalid record. Tutorial 2 distinguishes event delivery from business state. |
| S3 raw/curated/consumer layers, manifests, CSV/JSON/Avro/Parquet/ORC, compression, partitioning, small files | Tutorial 1 writes a manifest, creates an Athena CSV table, converts validated rows to partitioned Parquet, and compares query evidence. |
| Glue Catalog/crawlers/jobs/bookmarks/workflows/data quality, Spark transforms/joins/shuffles/skew | Tutorial 1 uses an explicit schema and supplies a Spark transformation plus rerun/bookmark reasoning. |
| Athena workgroups/results/encryption/scan limits/CTAS/partition pruning | Tutorial 1 provides SQL and query execution/verification procedures with empty-output-prefix and result-access checks. |
| Iceberg/Hudi/Delta, snapshots/merge/schema and partition evolution, compaction, expiration, S3 Tables | Tutorial 1 supplies an Iceberg update/delete extension and a writer/reader compatibility decision exercise. |
| IAM/S3/KMS/Glue/Lake Formation, LF-tags, row/column filters, cross-account, lineage/retention | Tutorial 1 verifies a constrained analyst's allowed and denied access and identifies direct-storage bypass paths. |
| Kinesis/MSK partitions/offsets/ordering/replay/lag, CDC snapshot+updates/deletes | Tutorial 2 executes duplicate/late/out-of-order cases locally and offers a bounded Kinesis publish/read experiment. |
| Flink event time/watermarks/windows/state/checkpoints/savepoints, Firehose delivery/errors/buffering | Tutorial 2 makes late-event behavior observable and maps state/sink guarantees to managed streaming choices. |
| Redshift Serverless/provisioned, fact/dimension grain, distribution/sort/encoding/statistics/materialized views/workload control | Tutorial 2 executes a local dimensional query and adapts it to a constrained Redshift warehouse load. |
| Step Functions, Glue workflows, MWAA/Airflow orchestration | Tutorial 1 defines data readiness/retry contracts; tutorial 2 includes an explicit bounded orchestration sequence and warns about persistent scheduler cost. |
| OpenSearch, Amazon Quick/QuickSight, Unified Studio/catalog/lakehouse | Tutorial 2 maps warehouse/search/dashboard consumers to distinct projections and authorization tests. Unified Studio integration also appears in [AI workspace tutorial](../15-ai-ml/18-accelerated-compute-and-unified-studio.md). |
| Performance, cost, controlled failure, recovery, teardown | Both require source-to-result reconciliation, query/stream metrics, and deletion of storage/results/versions and persistent compute. |

Prerequisites: SQL, Python, [S3](../../07-storage-and-backup.md), [IAM](../../04-iam-and-identity.md), and [database concepts](../../08-databases-and-caching.md). Use synthetic records only. No tutorial requires copying production data into a personal learning environment.
