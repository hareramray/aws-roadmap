# Turn an order-service brief into a defensible architecture

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial produces a completed design rather than a catalog of AWS services. No AWS resources are required. Keep a document named `order-platform-design.md` and record the calculations and decisions below with your own variations.

## 1. Replace vague requirements with measurable ones

The fictional business sells a small catalog internationally. It receives 864,000 read requests and 86,400 order-write requests per day. Peak traffic is assumed to be twenty times the daily average. Uploaded product images average 300 KB. New order records average 2 KB before indexes and replication. The team has three application engineers and no dedicated Kubernetes platform team.

Write these requirements:

| Operation | Required behavior | Verification |
| --- | --- | --- |
| Browse catalog | Return authorized current product data within a defined latency target | Load test with representative catalog size |
| Create order | Return one stable order ID for repeated use of the same idempotency key | Concurrent duplicate submission test |
| Pay | Never create two provider charges for one intended payment | Timeout-after-provider-acceptance test with a fake provider |
| Fulfill | Process accepted work asynchronously and expose status | Queue replay and reconciliation |
| Recover | Meet a chosen RTO/RPO for specified failures | Restore and failover evidence |

Assume a p95 API target of 400 ms for browsing and durable order acceptance. External fulfillment may take minutes and is measured separately. Use a fictional monthly availability objective and cost ceiling appropriate to the exercise, but label them requirements rather than service guarantees. Define sensitive fields and deletion requirements before choosing a storage model.

## 2. Calculate the first capacity estimate

Save `size_model.py`:

```python
reads_per_day = 864_000
writes_per_day = 86_400
peak_factor = 20
seconds_per_day = 86_400
mean_seconds = 0.15
order_bytes = 2_000
retention_days = 365
storage_multiplier = 3.0  # exercise assumption: indexes/metadata/copies

average_rps = (reads_per_day + writes_per_day) / seconds_per_day
peak_rps = average_rps * peak_factor
concurrency = peak_rps * mean_seconds
raw_order_bytes = writes_per_day * order_bytes * retention_days
print({"average_rps": average_rps, "peak_rps": peak_rps,
       "steady_state_inflight": concurrency,
       "raw_order_GB_decimal": raw_order_bytes / 1e9,
       "planning_storage_GB_decimal": raw_order_bytes * storage_multiplier / 1e9})
```

Expected results: average 11 requests/second, assumed peak 220 requests/second, roughly 33 in-flight requests at a 150 ms mean, 63.072 GB raw annual order data, and 189.216 GB using the explicitly assumed multiplier.

Do not size a concurrency limit from p95 as though it were the average in the steady-state relationship. Add burst, initialization, retry, and failure headroom. Replace the arbitrary storage multiplier with actual engine/index/replication measurements before purchasing capacity.

Perform sensitivity analysis. If mean latency rises to 600 ms at the same arrival rate, approximate in-flight work rises to 132. If each order request invokes four dependencies and retries one of them, dependency request rates exceed incoming order traffic. Write per-dependency rates rather than assigning 220 requests/second to every service indiscriminately.

## 3. Select compute using operational constraints

Compare EC2, ECS/Fargate, EKS, and Lambda using the same requirements. EC2 offers OS and hardware control but adds image/patch/fleet responsibility. ECS provides container orchestration with a relatively direct AWS operating model. EKS is justified when Kubernetes APIs and ecosystem needs outweigh platform work. Lambda suits bounded request/event work but requires concurrency, runtime, startup, and downstream-capacity design.

For this fictional team, select a modular application on ECS/Fargate or a small Lambda-based API after benchmarking; choose one for the final ADR. Here select Lambda for the intake API and queue worker because the work is bounded and bursty, while retaining separate modules in one repository. This does not imply every function must be a separate independently owned microservice.

Record why EKS was rejected: no Kubernetes-specific requirement and insufficient ownership capacity for its extra platform responsibilities. Record why EC2 remains a fallback: a future licensed native dependency or long-running process could require a different runtime. A valid ADR explains when its decision stops being valid.

## 4. Pick data stores by operation

Choose a relational database when transactional relationships, flexible queries, and established SQL are central. Choose DynamoDB when access patterns and partitioning fit its model. Both can be valid; “serverless” application compute does not force a particular database.

For this exercise, use a relational authoritative order store to make order/outbox atomicity straightforward. Product images use private object storage with controlled CDN delivery. Search, if required, is a derived index that can be rebuilt. Cache catalog reads only when measurement justifies it; the cache is not the source of truth for payment status.

Write the ownership table: orders owned by order application, images owned by catalog application, payment authority owned by the external provider with an internal reference, search index derived from catalog data. Specify backup, retention, recovery owner, and reconciliation for each.

## 5. Draw flows with contracts

Use a simple text flow and annotate every boundary:

```text
browser -> authenticated API -> order transaction + outbox
outbox publisher -> work queue -> fulfillment worker
worker -> payment adapter -> external provider
status reader -> authoritative order state
catalog image request -> CDN -> private object origin
```

Specify protocol, identity, timeout, retry owner, payload schema, and success meaning for each arrow. The queue carries a versioned event with order ID and correlation ID; large sensitive payloads remain in controlled storage. Public entry points require request validation, rate controls, and per-object authorization.

For the payment step, use a fake adapter during learning. Define a pending outcome when the provider response is ambiguous and a reconciliation operation to resolve it. Do not mark a timeout as a definite declined payment.

## 6. Build a realistic cost worksheet

Use AWS Pricing Calculator with current Region-specific rates. Enter API requests, compute duration/memory, database capacity, storage, backups, queue operations, CDN transfer, logging, tracing, and any network gateways/endpoints. Include idle baseline and temporary deployment/recovery capacity.

Calculate three scenarios: normal demand, a twenty-times average peak for an explicitly specified duration, and an error loop with a bounded retry count. A peak factor applied to every second of the month would overstate the expected case; omitting peak runtime entirely would understate it.

Write unit economics as `monthly attributable cost / successful orders`. Include failed attempts in the numerator because they consume resources, but do not count them as successful business outcomes. Compare a managed option with a self-managed option using operating effort and incident burden as well as cloud charges.

## 7. Complete the ADR and acceptance plan

Use this completed structure:

```text
Decision: Lambda-based intake and workers with an authoritative relational store.
Reason: Bounded bursty work, small team, clear transactional order/outbox boundary.
Alternatives: ECS service; EC2 fleet; EKS platform; DynamoDB authoritative store.
Risks: Connection bursts, external payment ambiguity, concurrency beyond database limits.
Controls: Bounded concurrency, suitable connection management, idempotency and reconciliation.
Evidence required: Representative load test, duplicate tests, restore test, cost estimate.
Revisit when: Runtime constraints, sustained workload economics, or platform requirements change.
Owner: Fictional order-platform team.
```

The architecture remains provisional until evidence is collected. Define tests for peak load, a slow database, a failed payment response, an invalid identity, and a deployment containing an incompatible schema change. A diagram does not prove any of these pass.

## Verification, troubleshooting, and cleanup

Have another learner recompute the 220 requests/second and explain every multiplier. Confirm each store has a recovery owner, each arrow has authentication and failure behavior, and the cost model includes networking and observability. Reject unbounded “auto scales” claims without quota and dependency analysis.

If the estimate appears too expensive, inspect idle services, assumed traffic duration, retention, and duplicated data before weakening security or recovery requirements. If latency fails, trace the slow path rather than changing every service simultaneously.

This design-only exercise has no AWS cleanup. Optional proof-of-concept deployments should use synthetic data, a capped runtime, dedicated resource identifiers, and an explicit teardown including retained databases, snapshots, logs, images, endpoints, and queues.

References: [AWS Architecture Center](https://aws.amazon.com/architecture/), [Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html), [AWS Pricing Calculator](https://calculator.aws/), [Builders' Library](https://aws.amazon.com/builders-library/).
