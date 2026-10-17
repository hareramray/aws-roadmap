# Database and caching tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../08-databases-and-caching.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — SQL, managed relational databases and recovery](01-sql-rds-aurora-and-dsql.md) | Executable SQL exercise and RDS/Aurora/DSQL decisions |
| [02 — DynamoDB correctness and access patterns](02-dynamodb-correctness-and-scaling.md) | Conditional-write tests, key/index design and consistency worksheet |
| [03 — Caches and specialized databases](03-caching-and-specialized-databases.md) | Staleness simulation and six specialized data-model exercises |

## Coverage mapping

| Source heading and named topics | Tutorial |
| --- | --- |
| Database selection/access patterns/transactions/latency/data size and alternatives | All three |
| Relational fundamentals: keys, normalization, constraints, indexes/plans, isolation/locks, migrations/connections | 01 |
| RDS: engines/versions/storage/subnets/parameters/options/maintenance/backups, IAM/SQL access, Multi-AZ variants/replicas/Proxy | 01 |
| Aurora: storage/compute/endpoints/failover, Serverless v2, Global Database, DSQL | 01 |
| DynamoDB: keys, Query/Scan, GSI/LSI, pagination/batches, conditions/transactions, TTL/Streams | 02 |
| DynamoDB capacity/hot keys/on-demand/provisioned/warm throughput, MREC/MRSC, DAX, backup/export/restore | 02, 03 |
| Cache-aside/invalidation/TTL/stampede/eviction/sharding/replicas/TLS/client behavior; ElastiCache Valkey/Redis OSS/Memcached, durability and MemoryDB | 03 |
| Specialized DocumentDB/Neptune/Keyspaces/OpenSearch/Redshift/time series | 03 |
| Implementation/monitoring/DMS/migration/restore workflow; relational lab | 01, 03 |
| Conditional-write lab and mastery | 02; verification in all |
