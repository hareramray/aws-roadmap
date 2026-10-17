# 08 — Databases, data models, consistency, and caching

[Roadmap index](README.md) · Prerequisites: [technical prerequisites](01-prerequisites.md), [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md), [storage and recovery](07-storage-and-backup.md)

Reviewed: **2026-10-18**. Begin with access patterns and correctness. A managed database removes infrastructure tasks, but schema design, authorization, query efficiency, capacity, retention, and recovery remain engineering responsibilities.

## Outcomes

Choose a database from workload requirements, design relational and DynamoDB access patterns, distinguish high availability from read scaling, handle connection and retry behavior, introduce caches safely, and validate backup restoration.

## 1. Choose by data and queries

Write representative operations before picking a database: create an order, list a customer's recent orders, conditionally reserve inventory, search descriptions, aggregate revenue, and expire a session. Specify transaction boundaries, latency, concurrency, record size, relationships, durability, geographic distribution, and consistency.

| Need | Candidate family | Main evaluation question |
| --- | --- | --- |
| Relational transactions and flexible SQL | RDS, Aurora | Engine compatibility, queries, connection/capacity model |
| Distributed relational transactions | Aurora DSQL | SQL feature compatibility, transaction behavior, regional topology |
| Key-value/document access by known keys | DynamoDB | Partition design and access patterns |
| Caching and fast data structures | ElastiCache | Eviction, persistence mode, consistency, client behavior |
| Durable in-memory database | MemoryDB or eligible durable ElastiCache configuration | Durability, topology, APIs, multi-Region requirements |
| Document-oriented compatibility | DocumentDB | Exact supported APIs and behavior, not brand-name equivalence |
| Graph relationships | Neptune | Traversal/query model and graph size |
| Cassandra-compatible workload | Keyspaces | API compatibility, partition design, throughput |
| Search and analytics indexes | OpenSearch Service | Index design, relevance, ingest, recovery of source data |
| Analytical warehouse | Redshift | Columnar analytics, loading, concurrency, separation from OLTP |

Time-series and specialized data services deserve separate evaluation when those access patterns dominate. Do not add multiple databases merely to use more services; each creates another consistency, backup, permission, and operations problem.

## 2. Relational foundations first

Understand primary/foreign keys, constraints, normalization, denormalization, indexes, transactions, isolation levels, locking, query plans, connection pooling, and migrations. A database engine cannot optimize a missing access pattern that your schema makes expensive.

Practice reading `EXPLAIN` output in your chosen engine. Measure query duration, examined rows, index use, lock waits, I/O, and connections. Avoid N+1 queries and unbounded result sets. Indexes accelerate selected reads but add storage and write overhead; an index on every column is not a strategy.

Use least-privilege database users or supported IAM authentication where it fits. IAM permission to describe an RDS instance does not grant SQL permission to read its tables. Encrypt connections, verify certificates, rotate credentials using supported workflows, and separate migration permissions from runtime permissions.

## 3. RDS deployment and operating choices

Study engine/version selection, instance class, storage type, storage autoscaling, subnet groups, parameter groups, option groups where applicable, maintenance windows, backups, monitoring, and deletion protection. Keep ordinary application databases private and allow only required application sources.

High availability and read scaling are distinct:

- **Multi-AZ DB instance deployment:** a primary plus a synchronous standby in another AZ; the standby does not serve read traffic. It is an availability feature.
- **Multi-AZ DB cluster deployment:** a writer and two readable standby instances in separate AZs for supported engines/configurations. This is a different deployment architecture.
- **Read replicas:** offload reads and can support other recovery or migration patterns, but replication, promotion, lag, and failover behavior depend on engine and configuration.

[RDS Multi-AZ instance deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html), [RDS deployment overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)

After failover, applications must reconnect and retry safe operations. Do not pin database endpoint IP addresses indefinitely. A retry after an ambiguous transaction result requires business-level idempotency or reconciliation; retrying an entire payment operation blindly is unsafe.

RDS Proxy can pool/manage supported database connections and help with connection bursts, but compatibility, transaction pinning, authentication, and cost must be evaluated. It is not a query optimizer or an automatic cure for too many simultaneous transactions.

## 4. Aurora and distributed relational options

Aurora has a cluster architecture with separate compute and distributed storage. Learn writer, reader, and instance endpoints; replica promotion; failover priorities; backups; engine compatibility; and read-after-write behavior when a request moves to a reader. A reader endpoint distributes connections; it does not split every SQL statement automatically.

Aurora Serverless v2 changes compute capacity within configured bounds. Study supported engine versions, minimum/maximum capacity, scaling responsiveness, connection pressure, and whether auto-pause is supported for the chosen configuration. The word “serverless” is not evidence that idle cost is always zero.

Aurora Global Database is relevant for cross-Region architectures; examine replication lag, switchover, failover, application routing, and region-specific configuration. A remote copy does not remove the need for a tested recovery procedure.

Aurora DSQL is a separate distributed relational service with PostgreSQL-compatible interfaces and active-active options. Study supported SQL features and transaction concurrency behavior before treating it as a drop-in PostgreSQL replacement. Region-set support and network round trips affect topology and latency. [Aurora DSQL overview](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/what-is-aurora-dsql.html)

## 5. DynamoDB data modeling

DynamoDB organizes items by partition key, optionally with a sort key. Design keys from access patterns and distribute load. A single popular partition key can be hot even when total table capacity appears sufficient. Randomizing keys can spread writes but may make reads expensive; document the tradeoff.

Example design exercise for an order-history workload:

| Operation | Possible key pattern | Follow-up concern |
| --- | --- | --- |
| Get one order | `PK=ORDER#id`, `SK=DETAILS` | Authorization by owner |
| List customer's orders | GSI partition `CUSTOMER#id`, sort timestamp/order ID | Pagination and index propagation |
| Store order line items | Same order partition, `SK=ITEM#id` | Item collection and transaction scope |
| Find a pending job | A deliberately designed status/time index | Hot status partition and sharding |

This is illustrative modeling, not a universal single-table prescription. Compare separate tables and single-table designs based on query coupling and team capability.

Learn `GetItem`, `Query`, `Scan`, conditional writes, update expressions, pagination, batch partial failures, transactions, TTL, Streams, and indexes. A filter expression is applied after data is read; it does not turn a broad scan into an efficient keyed query. TTL expiration is asynchronous and should not enforce precise access expiry; application logic must reject expired records when needed.

Global secondary indexes offer alternate key access and are eventually consistent. Tables and local secondary indexes support strong or eventual reads, while Streams and GSIs do not offer strong reads. Strong reads do not eliminate races between separate read and write operations; use conditions or transactions for invariants. [DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)

## 6. DynamoDB capacity and multi-Region behavior

Compare provisioned capacity with autoscaling against on-demand capacity. Understand read/write request sizing, consistency cost, transactions, index write amplification, throttling reasons, warm-throughput behavior, and configured limits. On-demand does not mean unbounded instantaneous capacity or zero cost risk.

Global tables support **multi-Region eventual consistency (MREC)** and **multi-Region strong consistency (MRSC)**. MREC uses asynchronous replication and conflict resolution; MRSC trades different latency and topology constraints for global strong consistency. Feature support differs: consult current restrictions for transactions, TTL, indexes, replica topology, and Region combinations before choosing. Do not teach that all global tables are necessarily eventually consistent. [Global table consistency modes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html)

Study PITR, on-demand backups, export/import, deletion protection, and restoration into a new table. Recreated tables need correct application configuration, permissions, alarms, and integrations; restored items alone do not restore the whole system.

## 7. Caching and durable in-memory data

Cache-aside reads check the cache, fetch on a miss, and populate with a TTL. Decide how writes invalidate or refresh data, whether stale reads are acceptable, and what happens when the cache is unavailable. Add TTL jitter, request coalescing, and bounded fallback to avoid a cache stampede overwhelming the primary database.

Study eviction policy, memory fragmentation, connection limits, hot keys, large values, serialization, sharding, replicas, failover, TLS, authentication, and client topology discovery. A high hit ratio can conceal one expensive uncached query; measure end-to-end latency and database load.

ElastiCache supports Valkey, Redis OSS, and Memcached configurations. Current ElastiCache for Valkey also offers eligible **durability-enabled** configurations, so the old blanket statement “ElastiCache is never durable” is incorrect. MemoryDB is another durable in-memory option, including supported multi-Region patterns. Compare the actual persistence mode and acknowledged-write guarantee. [ElastiCache and MemoryDB comparison](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/related-services-choose-between-memorydb-and-redis.html)

For durability-enabled ElastiCache, synchronous and asynchronous writes have different latency and possible data-loss behavior. A backup, replica, and synchronous durable log are different mechanisms. [ElastiCache durability options](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Durability.Options.html)

## 8. Implementation workflow and operations

Capture access patterns → choose correctness/consistency requirements → model keys/schema → test representative data volumes → secure connectivity and credentials → configure recovery → measure query/partition behavior → test failover and retries → document cost drivers.

Monitor latency percentiles, error/throttle rate, connections, CPU/memory, storage/IOPS, lock waits, replica lag, hot partitions, cache evictions, and backup failures as appropriate. For migrations, plan schema compatibility, change capture, validation, cutover, rollback, and application behavior during dual-running. AWS DMS can support migration workflows but does not eliminate semantic differences between engines.

## Lab A — DynamoDB conditional-write correctness

**Type:** sandbox lab using one small table, or use DynamoDB Local for an initial API exercise while recognizing its service differences.

1. Define an `Orders` table with string partition key `orderId`, using a dedicated lab name and chosen capacity mode.
2. Write a fabricated order with a condition that `orderId` does not already exist.
3. Repeat the same create request. Observe the conditional failure rather than silently overwriting the original.
4. Add a numeric version attribute and update only when it matches the version you read. Attempt a stale update.
5. Read using eventual and strong options, inspect consumed capacity, and test pagination with a small deliberate limit.

**Verify:** duplicate creation and stale updates are rejected; explain how the application converts these into safe business responses. **Cleanup/cost:** delete only the lab table, any indexes and explicit backups, and optional telemetry integrations. Requests, storage, backup, streams integrations, and diagnostics may be billable. Do not enable global tables just for this basic exercise.

## Lab B — Relational restore and failover plan

**Baseline:** use a local PostgreSQL database to create customers/orders, add a useful index, inspect a query plan, and demonstrate rollback of a transaction. Draft the equivalent private RDS configuration and recovery procedure.

**Optional paid AWS extension:** create a small private RDS test database with dummy data, supported backups, and a secure client path. Create a snapshot, restore to a separate test instance, and compare row counts plus selected record checksums. A Multi-AZ failover test is a separate optional expense; measure reconnection behavior if performed.

**Verify:** restored application queries work and no production endpoint was changed. **Cleanup:** remove both test instances, deliberate lab snapshots/backups, secrets, client compute, networking dependencies, and logs as appropriate. Stopping a database is not permanent teardown and does not eliminate all charges.

## Mastery checklist

- [ ] Choose a database from access patterns and correctness constraints.
- [ ] Distinguish RDS Multi-AZ instance, Multi-AZ cluster, and read replicas.
- [ ] Design an efficient DynamoDB query and conditional update.
- [ ] Explain GSI consistency and MREC/MRSC differences.
- [ ] Design cache failure behavior without overwhelming the database.
- [ ] Restore data and reconnect the application successfully.

Further references: [Aurora guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/), [DynamoDB guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/), [ElastiCache guide](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/), [AWS DMS guide](https://docs.aws.amazon.com/dms/latest/userguide/).
