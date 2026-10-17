# 20. Cost, FinOps, and performance

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete [account setup](02-account-and-tooling.md), core compute/storage/database modules, [observability](16-observability-and-operations.md), and [architecture](18-architecture-and-system-design.md). Familiarity with percentages, averages, percentiles, and spreadsheet-style calculations is sufficient.

Learn to estimate costs before deployment, attribute actual spend, detect waste, compare pricing commitments, and optimize performance using measurements. FinOps is an ongoing collaboration between engineering, finance, and business owners. Its objective is useful business value per unit of spending, not the smallest possible bill regardless of service quality.

## 1. Understand the shape of an AWS bill

Prices vary by service, Region, purchasing option, resource configuration, usage dimension, and time. Some resources incur charges while idle; others charge by request, bytes processed, storage duration, or execution time. Taxes, support, Marketplace subscriptions, and negotiated agreements can affect totals.

Build an inventory of cost drivers:

| Area | Dimensions to inspect |
| --- | --- |
| Compute | Running duration, instance/task size, architecture, licenses, accelerators, commitments |
| Serverless | Requests, execution duration, memory, concurrency-related features, integrations |
| Storage | Capacity, storage class, operations, retrieval, minimum duration/size rules, replication |
| Databases | Compute/capacity, I/O, storage, backups, replicas, data transfer |
| Networking | Transfer direction, NAT processing, gateways, endpoints, load balancers, public IPv4 |
| Observability | Ingestion, retention, custom metric cardinality, queries, traces, dashboards |
| Analytics and AI | Bytes scanned, processing time, tokens, model hosting, vector storage, data preparation |
| Governance and security | Recorded resources, events, scans, findings, log storage, account coverage |

Avoid treating a headline instance price as the application's total cost. A small application can spend more on network gateways, load balancing, telemetry, or an idle database than on its business compute.

Stopping an instance does not remove attached storage, snapshots, addresses, or related infrastructure. Deleting a stack can leave retained resources. Stopping a scheduled workload does not necessarily cancel an external software subscription. Cost review must inspect both resource inventory and billing evidence.

## 2. Estimate before building

Use AWS Pricing Calculator with the intended Region and architecture. Save assumptions alongside the estimate: operating hours, traffic, payload sizes, storage growth, retention, replication, logging volume, and peak-to-average ratio. Distinguish fixed baseline costs from demand-driven costs.

```text
estimated monthly cost
  = baseline capacity and subscriptions
  + usage-driven requests, compute, and processing
  + storage, backups, and retention
  + networking and transfer
  + observability, security, and support
```

Create low, expected, and high scenarios. Model an unexpected traffic spike and a misconfigured loop, because retry storms, recursive event flows, high-cardinality metrics, and unlimited data scans can increase spend quickly. Free Tier credits or promotional discounts are not a substitute for a sustainable steady-state estimate.

Estimate total cost of ownership for architectural comparisons. Include engineering time, upgrades, incident response, backup operations, licensing, migration, and exit effort. A managed service can cost more per resource while reducing total operating work. A cheaper instance can be more expensive per successful job if it takes much longer.

## 3. Put visibility and accountability in place

Begin with Cost Explorer for interactive trends and service/account breakdowns. Use cost allocation tags and Cost Categories to connect technical usage to teams, products, environments, and customers where appropriate. Standardize tags such as owner, application, environment, cost center, and expiry date, while recognizing not all charges are directly taggable.

Define an allocation policy for shared costs. Shared networking might be allocated by traffic, platform infrastructure by tenant count, and security services by account or usage. State whether the policy is showback, which informs owners, or chargeback, which affects internal budgets. Arbitrary allocation can distort engineering incentives.

Learn AWS Data Exports and CUR 2.0 for detailed cost and usage analysis. AWS recommends CUR 2.0 for detailed billing exports; inspect the actual schema and refresh behavior before building queries. [AWS Data Exports overview](https://docs.aws.amazon.com/cur/latest/userguide/what-is-data-exports.html).

Understand unblended, amortized, net, and other cost views. A commitment's upfront payment and its allocation across usage can make different views look very different without either being wrong. Document the view used in every report, including credits, refunds, taxes, and support treatment.

Treat billing exports as sensitive business data. Restrict access to account identifiers, usage patterns, contract-related information, and customer allocation. Configure storage retention and query limits; analyzing a bill can itself create S3, Athena, dashboard, or other service charges.

## 4. Use budgets and anomaly detection correctly

Set actual and forecast budgets at account and workload levels. Choose thresholds that allow time to respond, and verify notification delivery. Combine them with Cost Anomaly Detection, resource quotas, deployment restrictions, concurrency limits, expiry automation, and bounded experiments.

**AWS Budgets is not a hard spending cap.** Billing and notifications are delayed, and additional charges can accrue before an alert or action runs. Budget actions can restrict selected activities or trigger supported responses, but they do not universally stop all AWS usage, erase existing charges, or cancel commitments. [AWS Budgets behavior and delays](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html).

Keep AWS Budgets separate from account-plan features. The current Free Tier FAQ describes project spend limits and pausing behavior in the newer signup experience. Eligibility and scope depend on the account/project experience; those controls do not change the behavior of an ordinary AWS Budget. Check the live terms for your account instead of assuming either that all AWS spending is capped or that no account-specific spending control exists. [AWS Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/).

A useful response playbook identifies the account, service, Region, usage type, recent deployment, and owner. It then selects a safe mitigation. Automatically stopping a production database because a budget threshold was crossed may cause more harm than the charge. For sandboxes, carefully scoped expiry and resource-creation restrictions are often appropriate.

Forecasts require context. Seasonal traffic, a migration's double-running period, a new product, or a one-time backfill can look anomalous. Review unusual spend with the business owner before labeling it waste.

## 5. Optimize in the right order

1. **Remove waste:** unused volumes, abandoned load balancers, obsolete snapshots, idle clusters, unused development environments, forgotten endpoints, and unnecessary data copies.
2. **Schedule predictable inactivity:** stop or scale down suitable nonproduction capacity while preserving required data and startup procedures.
3. **Rightsize:** match CPU, memory, storage throughput, IOPS, network, and accelerator capacity to measured needs.
4. **Improve architecture and code:** reduce queries, payloads, serialization, scans, retries, duplicated work, and unnecessary synchronization.
5. **Select efficient purchasing options:** purchase commitments only after demand and architecture are sufficiently understood.

Use Compute Optimizer and Cost Optimization Hub as decision support. Validate recommendations against memory metrics, peak events, licensing, recovery capacity, latency objectives, and future plans. A low CPU average does not prove that a database or memory-intensive service can be safely reduced.

Prioritize changes by expected savings, implementation effort, confidence, and risk. Assign an owner and verify realized savings after deployment. Avoid claiming all displayed recommendation savings are additive when alternatives overlap.

## 6. Compare purchasing models

| Model | Suitable use | Main caution |
| --- | --- | --- |
| On-Demand | New, uncertain, or temporary workloads | Higher unit price may be worthwhile for flexibility |
| Savings Plans | A measured baseline of eligible usage matching the plan's terms | Commitment can outlast a workload or architecture |
| Reserved Instances | Eligible predictable service usage with appropriate scope | Benefits and flexibility differ by service and reservation type |
| Spot | Interruption-tolerant jobs, flexible fleets, suitable workers | Capacity and interruptions require engineering support |
| Capacity Reservations | Workloads requiring specific capacity assurance under supported terms | Capacity assurance and price discounts are different concerns |

Understand **coverage** versus **utilization**. Coverage asks how much eligible usage receives a commitment benefit; utilization asks how much purchased commitment is consumed. Maximizing coverage without studying utilization can lock in unnecessary spend.

Do not buy commitments as a learning exercise. Model them using historical demand and compare term, payment option, eligible services, scope, sharing behavior, and flexibility. Discounts do not automatically reserve capacity; inspect the specific product terms. Spot interruptions require checkpointing, graceful shutdown, retry-safe work, and sufficient alternative capacity.

## 7. Tune performance scientifically

Start with an objective: for example, reduce p95 checkout latency while maintaining correctness and acceptable cost. Establish a baseline, identify the bottleneck, change one meaningful variable, and repeat the same workload. Record the result and rollback condition.

Investigate the whole path: client/network latency, connection setup, TLS, load balancer, application CPU, memory allocation, database queries, indexes, locks, storage latency, and external dependencies. Vertical scaling can hide inefficient queries temporarily; adding more application instances can make a database bottleneck worse.

Measure throughput and tail latency under saturation. A service may appear fast at low load but collapse when queues grow. Use appropriate connection pools, caching, batching, compression, partitioning, asynchronous work, and autoscaling signals. Every optimization has correctness or freshness implications that need testing.

Compare processor architectures, including Graviton where compatible, with your actual binaries and dependencies. Benchmark price per completed unit of work, not price per vCPU alone. Include startup time, data loading, failure/retry rate, and operational effort in job economics.

## 8. Connect efficiency to sustainability

Reduce idle resources, eliminate unnecessary data retention, improve utilization, and move fewer bytes. Select efficient managed services and instance families based on measured workload behavior. Use sustainability data and current regional requirements where available, but avoid claiming a precise carbon reduction from a cost reduction alone.

Security, legal retention, availability, and residency requirements still apply. Deleting required backups to reduce storage or consolidating all capacity into one failure domain can damage the business. Sustainability review belongs alongside the other architecture pillars.

## Practical lab: optimize a small workload with evidence

**Scope:** choose a disposable API, batch job, or data query from an earlier module. Use synthetic data and a fixed maximum runtime.

1. Write an expected cost model with current prices and a bounded experiment size.
2. Tag the workload and enable the appropriate billing visibility. Record the cost view you will use.
3. Run a baseline workload, capturing success rate, throughput, latency, resource saturation, and units processed.
4. Identify one likely inefficiency, such as an unbounded query, oversized memory allocation, missing cache, or excess logging.
5. Apply one change and repeat the same workload. Confirm the output is equivalent and error rates have not increased.
6. Calculate cost per successful request, completed job, or processed gigabyte. Separate measured resource usage from estimated currency cost when billing has not yet arrived.
7. After billing data is available, compare the estimate with actual spend and explain differences.

**Verification:** produce a before/after table with workload assumptions, unit cost, tail latency, throughput, and correctness evidence. Keep a change only if its tradeoff meets the objective. A faster result that silently drops records is a failed optimization.

**Cleanup:** stop load generation, delete temporary workloads, retained logs, dashboards, export/query artifacts, snapshots, endpoints, and scheduled jobs that are no longer needed. Recheck relevant Regions and billing later because charges are not instantaneous. Keep budget notifications active if the account remains in use.

## Pitfalls and mastery

- Treating a budget alert as a guaranteed spending ceiling.
- Optimizing a percentage discount instead of total or unit cost.
- Purchasing commitments before removing idle capacity.
- Ignoring networking, observability, license, and backup costs.
- Comparing estimates built for different load, Regions, or reliability assumptions.
- Reducing capacity without testing peak traffic and failure headroom.

- [ ] I can explain the largest cost drivers of my workload.
- [ ] I can produce an estimate with explicit assumptions and uncertainty.
- [ ] I can distinguish coverage, utilization, capacity assurance, and discounts.
- [ ] I can attribute shared costs using a documented policy.
- [ ] I can respond to an anomaly without blindly disrupting production.
- [ ] I can demonstrate a performance/cost improvement with equivalent outputs.

## Official references

- [AWS Pricing Calculator](https://calculator.aws/)
- [AWS Cost Management documentation](https://docs.aws.amazon.com/cost-management/)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/)
- [Cost Optimization Hub](https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization-hub.html)
- [Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Performance Efficiency Pillar](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/welcome.html)

Continue to [migration, hybrid, and edge](21-migration-hybrid-and-edge.md).
