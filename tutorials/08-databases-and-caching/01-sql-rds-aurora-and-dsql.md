# Tutorial 08.1 — Build relational invariants and a managed-database recovery plan

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../08-databases-and-caching.md)

**Mode:** executable SQL in a disposable PostgreSQL database; optional private RDS deployment; Aurora/DSQL topology exercises. **Result:** query and transaction evidence plus a restore plan. Reviewed 2026-10-18.

## 1. Define the operations

The portal sells fictional processing credits. It needs to create a customer, create an order, list recent orders and prevent negative quantities. Record whether related changes must commit together. Flexible relational queries and explicit constraints make SQL a reasonable baseline; choosing it does not imply every future search/analytics operation belongs on the primary database.

Use an already installed local PostgreSQL server or an approved disposable database. Do not connect these exercises to a real application schema. SQL below uses the dedicated schema `roadmap_sql`.

```sql
CREATE SCHEMA roadmap_sql;
CREATE TABLE roadmap_sql.customers (
  customer_id integer PRIMARY KEY,
  display_name text NOT NULL
);
CREATE TABLE roadmap_sql.orders (
  order_id integer PRIMARY KEY,
  customer_id integer NOT NULL REFERENCES roadmap_sql.customers(customer_id),
  quantity integer NOT NULL CHECK (quantity > 0),
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO roadmap_sql.customers VALUES (1, 'Alice'), (2, 'Bob');
INSERT INTO roadmap_sql.orders(order_id, customer_id, quantity)
VALUES (101, 1, 3), (102, 1, 1), (103, 2, 2);
SELECT order_id, quantity FROM roadmap_sql.orders
WHERE customer_id = 1 ORDER BY created_at DESC, order_id DESC;
```

Expected: Alice's two orders, not Bob's. Foreign keys express relationships; application authorization must still ensure Alice cannot request Bob's rows just by changing an ID.

## 2. Test invariants and transactions

Attempt an order with quantity `-1`, then one with unknown customer `999`. Each should fail a constraint. Run these as separate test statements so a transaction left in an aborted state does not confuse later results.

```sql
BEGIN;
UPDATE roadmap_sql.orders SET quantity = 5 WHERE order_id = 101;
ROLLBACK;
SELECT quantity FROM roadmap_sql.orders WHERE order_id = 101;
```

Expected quantity remains `3`. Discuss isolation: two clients reading then writing can conflict. Use supported locking, conditions and transaction patterns for the invariant; do not assume “SQL transaction” means every isolation level prevents every anomaly. Keep transactions short and inspect lock waits during diagnosis.

Normalization separates customers from orders and avoids repeating customer attributes. Denormalization can help selected reads but requires an update/consistency plan. An ORM does not remove N+1 query risks or the need to understand generated SQL.

## 3. Inspect a query plan

```sql
CREATE INDEX orders_by_customer_time
ON roadmap_sql.orders(customer_id, created_at DESC, order_id DESC);
EXPLAIN SELECT order_id, quantity FROM roadmap_sql.orders
WHERE customer_id = 1 ORDER BY created_at DESC, order_id DESC LIMIT 20;
```

With only three rows, PostgreSQL may legitimately choose a sequential scan. Do not claim the index is broken because it is unused on tiny data. Use representative generated synthetic rows if studying optimizer behavior, run statistics maintenance as appropriate, and compare examined rows, timing and write overhead.

Parameterize application queries. Indexes improve particular access patterns but add write/storage costs. An index on every column is not a substitute for a workload model.

## 4. Map the design to RDS

For an optional AWS extension, choose a supported PostgreSQL version and small suitable instance/storage configuration after pricing. Use private database subnets across the required AZs, a database group accepting only the lab client's group, encryption, backup retention and intentional maintenance windows. Connect from a secure client path such as a dedicated managed EC2 instance, not a globally open database port.

Record DB subnet group, parameter group, engine version, storage/IOPS, monitoring, credentials/secret ownership and deletion protection. Option groups apply to relevant engines; do not assume every engine has identical features. IAM permission to manage RDS is distinct from SQL user grants. Use a limited runtime database user and a separate migration identity.

Require TLS with certificate validation using the current RDS CA bundle and client guidance. Supported IAM database authentication can replace some password use but still needs engine/user setup and application compatibility. Never place a password in shell history or commit a connection string containing it.

## 5. Choose availability and connection behavior

Compare a Multi-AZ **DB instance** with one standby that does not serve reads, a supported Multi-AZ **DB cluster** with writer and readable standbys, and separate read replicas. State whether the objective is availability, read throughput or geographic recovery. A replica's lag can make a recent write invisible to a read routed there.

Write a connection-failure test: interrupt the test connection or perform an authorized lab failover, observe reconnect behavior and retry only business-safe operations. DNS endpoints can change underlying addresses; applications must not cache addresses indefinitely.

Evaluate RDS Proxy for connection bursts. Record engine support, pooling behavior, transaction/session pinning and authentication. It does not repair an inefficient query or make unlimited concurrent transactions safe. Monitor active connections, query latency, locks, CPU, memory, storage and replica lag as applicable.

## 6. Work Aurora and DSQL alternatives

For Aurora, draw compute instances over the cluster's storage, writer endpoint, reader endpoint and instance endpoints. Route writes intentionally; a reader endpoint distributes connections and does not inspect every SQL statement to decide where it goes. Define promotion/failover priorities and test client behavior.

For Serverless v2, choose minimum/maximum capacity and record version-specific scaling/auto-pause support before estimating idle cost. For Aurora Global Database, draw primary/secondary Regions, replication, routing and controlled switchover/failover. Measure lag and recovery instead of assuming the second Region proves RPO zero.

For Aurora DSQL, take the `customers/orders` model and make a feature-compatibility checklist: supported types, constraints, transaction behavior, concurrency conflicts, connection/authentication and Region topology. Do not run the PostgreSQL schema unchanged without checking supported SQL. Design a retry of a conflicting transaction only when the business operation remains idempotent. DSQL's PostgreSQL-compatible interface does not guarantee every PostgreSQL extension or engine behavior.

## 7. Restore and migrate with evidence

In optional RDS, create a dedicated snapshot and restore to a separate private test instance. Reapply/check access and configuration, connect to the new endpoint and compare selected rows and invariants. Record elapsed time and potential data loss. Do not overwrite a production endpoint for a learning test.

For a migration design, use source inventory → schema/feature assessment → initial load → change capture if appropriate → validation → cutover → rollback. DMS can assist supported movement; schema conversion and semantic compatibility remain separate work. Add a schema change during migration and decide how it is coordinated.

## Verify, troubleshoot, and clean up

Pass with constraint failures, rollback evidence, an explained query plan and a validated restore or clearly labeled recovery design. If queries fail after migration, inspect schema/search path, grants, encoding/time zones and unsupported features before blaming network transport.

**Cleanup/cost:** drop only `roadmap_sql` from the disposable database after reviewing its contents. Delete dedicated AWS test/restore instances and selected lab snapshots, secrets, clients and logs. Stopping RDS is not permanent teardown and storage/backups can remain billable.

Sources: [RDS guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/), [Aurora guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/), [Aurora DSQL](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/what-is-aurora-dsql.html), [DMS](https://docs.aws.amazon.com/dms/latest/userguide/).
