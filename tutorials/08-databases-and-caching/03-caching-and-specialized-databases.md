# Tutorial 08.3 — Demonstrate cache staleness and choose specialized data models

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../08-databases-and-caching.md)

**Mode:** local cache simulation and six worked specialist designs; no managed database fleet required. **Result:** invalidation/failure behavior plus a justified data-service comparison. Reviewed 2026-10-18.

## 1. Simulate cache-aside

Use PowerShell to represent one authoritative record and one cached copy:

```powershell
$sourceRecord = @{ Price = 20; Version = 1 }
$cacheRecord = @{ Price = $sourceRecord.Price; Version = $sourceRecord.Version }
$sourceRecord.Price = 25
$sourceRecord.Version = 2
[pscustomobject]@{ DatabasePrice = $sourceRecord.Price; CachedPrice = $cacheRecord.Price }
```

Expected: database 25, cache 20. The cache did not become current because the source changed. Define a read flow: check cache, fetch from source on miss, populate with an expiry, and return. Define a write flow: update source and invalidate or refresh the cache according to the consistency requirement.

Now remove the cached entry and refill from the source. Expected cached price becomes 25. This toy model demonstrates staleness only; it is not a distributed cache implementation or proof of atomic source/cache updates.

## 2. Handle failures and a stampede

Suppose 1,000 requests arrive after a popular key expires. If all fetch the database independently, a cache miss becomes a traffic spike. Design request coalescing for one in-flight refresh, bounded concurrency and TTL jitter. Decide whether stale data can be served while refreshing for this particular operation.

For a product description, short stale reads may be acceptable. For authorization or final purchase price, use the authoritative correctness rule; do not let a stale cache silently become the security or payment decision.

Simulate cache unavailability by bypassing it. Bound fallback so the primary database is not overwhelmed. Define timeout/deadline and retry ownership. A cache failure should produce a known degraded mode, not an unlimited retry storm.

## 3. Select an in-memory configuration

Compare ElastiCache Valkey/Redis OSS and Memcached configurations using required data structures, sharding, replicas, failover, client compatibility and operations. Map strings for cached JSON, sets for a membership use case, and sorted structures for a leaderboard only where the chosen engine supports the needed semantics.

Record eviction policy, memory headroom/fragmentation, large values, connection pooling, hot keys, topology discovery, TLS and authentication. A serverless cache and a node-based cluster have different sizing/operation considerations; verify current feature support instead of copying one configuration across both.

Current eligible ElastiCache for Valkey configurations can enable durability. Compare synchronous and asynchronous write modes by acknowledged-write guarantee and latency. MemoryDB is another durable in-memory candidate, including supported multi-Region patterns. Do not say every ElastiCache configuration is disposable or every replica prevents acknowledged-write loss.

Write three acceptance cases: cache node lost, whole cache unavailable, and authoritative durable-store failure. Specify what data may be lost in the chosen mode and how the client recovers. Backups, replicas and a durable transactional log are separate mechanisms.

## 4. Document-model exercise

Create three synthetic customer profile documents containing ID, name and an array of contact preferences. The application needs full-profile retrieval and a query for customers opting into a particular category. For DocumentDB, design an appropriate document/index structure and verify the exact MongoDB-compatible API/version behavior required by the client.

Negative case: a query depends on an unsupported operator or transaction behavior. Record a compatibility test before migration, rather than assuming a familiar driver name guarantees identical engine semantics. Compare storing the same profile in PostgreSQL JSON or DynamoDB when those existing services already meet the access pattern.

The deliverable is a sample document, two queries in the selected supported syntax, an index plan and a rejection reason for alternatives. It is a design exercise until you validate those operations against the actual chosen engine.

## 5. Graph and wide-column exercises

For Neptune, draw `Alice → purchased → BookA`, `Bob → purchased → BookA`, and `Bob → purchased → BookB`. Ask for books bought by people who bought Alice's book. Identify vertices/edges, choose a supported graph query model and write the traversal in plain steps before translating it to that model's query language. Count expected output: BookB is the candidate after excluding already purchased items.

Negative case: one highly connected product creates a huge expansion. Bound traversal depth/results and measure query work. A graph service is justified by relationship traversal needs, not by data containing any relationships at all.

For Keyspaces, model device readings by a partition such as `(device_id, day)` and clustering timestamp. Write the expected query “device-7 readings for one day ordered by time.” Then request “all devices above a temperature across a year”; explain why that different access pattern needs another data path. Check Cassandra-compatible API/driver/authentication behavior and partition distribution.

## 6. Time-series and search exercises

For a time-series workload, create ten synthetic measurements with timestamp, device tag and temperature. In a Timestream for InfluxDB evaluation, define retention, timestamp precision, tag cardinality and a supported query for a five-minute mean. Compare hot recent data with long-term archive/analytics. Do not confuse this product with maintenance-stage Timestream for LiveAnalytics.

For OpenSearch, define documents with `title`, `body`, `tenant_id` and `created_at`. Choose text versus exact-match field mappings deliberately. Search for a phrase within one tenant, then test that another tenant's result cannot leak through a missing filter or authorization rule. Define source-of-truth storage and a rebuild procedure: a search index is often derived data, not the only authoritative copy.

For vector search, compare the service's supported vector capabilities with S3 Vectors or a relational vector option using recall, latency, filters, update rate and cost. Use one fixed embedding model/dimension and a small labeled relevance set. Semantic similarity does not itself establish authorization.

## 7. Warehouse exercise and migration decision

For Redshift, sketch a fact table of orders and customer/date dimensions. The query sums monthly revenue across many rows; this differs from a transactional “update one order” workload. Define loading cadence, schema evolution, query concurrency and separation from the OLTP database.

Use three fabricated orders to calculate the expected monthly total manually. Your analytical pipeline should reproduce that total and identify duplicates. A fast dashboard with double-counted events fails correctness. Compare a warehouse with Athena over appropriately formatted/partitioned files when the query frequency and operational requirements differ.

For each specialist candidate, complete access patterns, consistency, IAM/database authorization, network, encryption, backups, metrics, unit costs and migration/exit plan. Use DMS only for supported movement and validate source/target semantic differences independently.

## Verify, troubleshoot, and finish

Pass with a demonstrated stale cache, a safe fallback design and six model/query worksheets whose expected results are known. If a query is slow, inspect access pattern, key/index, cardinality and client concurrency before adding nodes. If results are wrong, check duplicate handling, schema mapping, time zones and filters.

**Cleanup/cost:** local hashtables disappear with the session. No managed specialist cluster is required. Optional deployments need explicit teardown of nodes/serverless resources, backups, endpoints, secrets and logs; retained storage and minimum capacity can continue charging.

Sources: [ElastiCache/MemoryDB comparison](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/related-services-choose-between-memorydb-and-redis.html), [durability modes](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Durability.Options.html), [DocumentDB](https://docs.aws.amazon.com/documentdb/latest/devguide/what-is.html), [Neptune](https://docs.aws.amazon.com/neptune/latest/userguide/intro.html), [Keyspaces](https://docs.aws.amazon.com/keyspaces/latest/devguide/what-is-keyspaces.html), [Timestream for InfluxDB](https://docs.aws.amazon.com/timestream/latest/developerguide/timestream-for-influxdb.html), [OpenSearch](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html), [Redshift](https://docs.aws.amazon.com/redshift/latest/dg/welcome.html), [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
