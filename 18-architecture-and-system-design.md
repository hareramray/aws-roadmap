# 18. Architecture and system design

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete the core networking, compute, storage, database, security, integration, deployment, observability, and reliability modules. You do not need expert knowledge of every AWS product; you need enough understanding to compare realistic alternatives and identify what remains uncertain.

By the end, you should turn an ambiguous business request into measurable requirements, select a defensible architecture, document tradeoffs, estimate capacity and cost, and review the result using the AWS Well-Architected Framework. A good architecture explains how a system will behave under load, failure, change, and misuse—not just where service icons connect.

## 1. Translate the problem into requirements

Start with users, workflows, data, and business consequences. For an order service, identify browsing, inventory checks, payment authorization, order confirmation, fulfillment, cancellation, and support. Identify external systems and which team owns each boundary.

Write functional requirements separately from quality requirements:

- **Function:** a customer submits an order and receives an order identifier.
- **Correctness:** an accepted order cannot be silently lost or charged twice.
- **Performance:** define throughput, concurrent users, payload sizes, and latency percentiles.
- **Availability:** define the measurement window and what constitutes a usable response.
- **Recovery:** specify RTO, RPO, and disaster scenarios.
- **Security:** define identities, trust boundaries, sensitive fields, and allowed access paths.
- **Operations:** specify ownership, deployment frequency, troubleshooting, and maintenance expectations.
- **Economics:** define a budget range and a cost-per-successful-operation goal.
- **Constraints:** existing technology, skills, contracts, licensing, geography, and deadlines.

Document assumptions and uncertainty. “A million users” is not a useful load model until you know active users, arrival patterns, reads versus writes, session length, and payload size. Distinguish current demand from planned demand and a short-lived marketing spike.

## 2. Apply the six Well-Architected pillars

AWS defines six pillars: operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability. Use them throughout design rather than as a final compliance checklist. [AWS Well-Architected pillars](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html).

| Pillar | Questions to ask of your design |
| --- | --- |
| Operational excellence | Who owns it? Can deployments, incidents, and improvements be performed consistently? |
| Security | Are identities constrained, data protected, and security events detectable and actionable? |
| Reliability | What fails independently? Can the workload recover and preserve correct behavior? |
| Performance efficiency | Does the chosen resource match the workload, and has that been measured? |
| Cost optimization | Which work creates value, which capacity is wasted, and who sees the cost? |
| Sustainability | Can you reduce unnecessary computation, storage, transfers, and idle resources? |

Tradeoffs must be explicit. A second Region can improve recovery coverage while increasing cost, data complexity, and resource use. Reducing telemetry may save money while weakening incident diagnosis. Record the requirement driving the choice and the evidence that the chosen compromise is acceptable.

Learn the Well-Architected Tool, workload reviews, milestones, lenses, and improvement planning. A review surfaces risks and decisions; it does not certify that the application is secure or guarantee an availability level. Prioritize improvements by customer consequence and feasibility.

## 3. Estimate scale before selecting capacity

Use a simple, transparent workload model:

```text
average request rate = daily requests / seconds per day
peak rate = average rate × assumed peak factor
rough in-flight concurrency = request rate × average response time in seconds
daily raw storage = new records per day × average record bytes
```

The concurrency relationship is a useful steady-state approximation, not a substitute for burst tests. Add replicas, indexes, logs, backups, metadata, and retention to storage estimates. Include retries and downstream fan-out when estimating dependency traffic. If one request invokes five services, the downstream request count may greatly exceed the incoming count.

Estimate network volume and direction. Cross-AZ, cross-Region, internet egress, NAT processing, and service endpoints can materially change costs and performance. Check quotas and available capacity for the selected Region and account. Limits can apply to APIs, connections, partitions, concurrency, instance counts, IP space, and deployment rates.

Use load tests to replace guesses. Record data shape, warm versus cold state, success rate, p95/p99 latency, saturation, and cost. Stop the test at a bounded budget or resource ceiling and use synthetic data.

## 4. Choose compute and application boundaries

| Approach | Strong fit | Tradeoffs to evaluate |
| --- | --- | --- |
| EC2 | OS-level control, specialized software, legacy hosting | Patching, scaling, images, and fleet operations |
| ECS with an appropriate capacity model | Containers with managed orchestration and AWS integration | Task sizing, networking, deployments, and workload isolation |
| EKS | Kubernetes ecosystem requirements or established platform skills | Cluster/platform complexity, upgrades, policy, and add-ons |
| Lambda | Event-driven and request-driven work fitting runtime constraints | Concurrency, duration, startup, packaging, and downstream limits |
| Managed purpose-built service | A standard capability such as a database or queue | Service semantics, supported features, portability, and pricing |

A modular monolith may be the best initial design for a small team. Microservices create independent deployment and scaling boundaries but add distributed transactions, network failures, schema compatibility, and operational ownership. Choose boundaries around business capabilities and change patterns rather than creating one service for each database table.

Separate public API contracts from internal implementation. Define versioning, authentication, authorization, validation, pagination, rate limiting, idempotency, error formats, and compatibility guarantees. An API gateway or load balancer cannot infer the correct business authorization for every object.

## 5. Model data and consistency explicitly

Start with access patterns and invariants. Relational databases suit transactional relationships and flexible queries; key-value/document models suit known access patterns and suitable partitioning; object storage suits durable objects and large analytical datasets; search indexes suit retrieval but are often derived rather than authoritative state.

Decide who owns each record, where transactions begin and end, and what consistency each operation requires. A shopping recommendation can tolerate stale data more easily than a payment balance. Document partition keys, hot-key risks, indexes, write amplification, retention, and schema migration.

Study caching: cache-aside, write-through, expiration, invalidation, stampedes, negative caching, and stale reads. A cache should have a defined failure behavior. A cached authorization decision or inventory count may have different correctness implications from a cached public product description.

For distributed workflows, understand sagas, compensating actions, the transactional outbox, CDC, idempotent consumers, deduplication, and reconciliation. Compensation is a new business action, not literal time travel. Refunding a payment does not erase the fact that a charge happened.

## 6. Design asynchronous flows intentionally

Use queues to buffer work, absorb bursts, and separate producer and consumer availability. Use pub/sub when multiple consumers need independent delivery. Use event buses for routing integration events, and streaming platforms when ordered event streams, replay, or multiple stream processors are requirements.

Document delivery semantics, ordering scope, duplicate handling, retention, poison-message behavior, dead-letter handling, and backpressure. “Exactly once” claims must be examined at the business effect boundary: a broker's delivery feature does not automatically make a payment API or database mutation happen exactly once.

Distinguish commands from events. “Charge this order” requests work; “Order accepted” reports a past fact. Define schemas and evolution rules, avoid placing secrets in event payloads, and ensure replay cannot repeat harmful side effects. Track queue age and business completion latency, not only queue length.

## 7. Cross-cutting design concerns

Create a threat model with identities, entry points, trust boundaries, sensitive stores, and plausible abuse paths. Apply least privilege, encryption, secrets management, validation, rate limits, and audit evidence. Include software supply-chain controls and dependency updates.

For multi-tenancy, compare shared infrastructure, tenant-specific partitions, separate databases, and separate accounts. Evaluate isolation, noisy-neighbor effects, encryption boundaries, per-tenant quotas, operational burden, and cost allocation. Authentication alone does not provide tenant isolation; every access path needs a tenant-aware authorization design.

Define deployment behavior: schema compatibility, progressive traffic, rollback, feature flags, and recovery from partial rollout. Treat background workers and scheduled jobs as part of the release. A database migration can make an application rollback unsafe even when reverting a container image is easy.

Plan graceful degradation. Decide which functionality can be temporarily disabled, served from stale data, queued, or handled manually. Include external providers in the availability model rather than assuming AWS services are the entire dependency graph.

## Practical design exercise: an order-processing platform

**Scope:** create an architecture document and optionally implement a tiny slice in a sandbox. The design itself incurs no AWS charges.

1. State assumed peak request rate, read/write mix, payload size, availability target, RTO, RPO, team size, and budget range.
2. Draw the request path and asynchronous fulfillment path. Show public/private boundaries, accounts, Regions, Availability Zones, and data stores.
3. Compare two viable compute choices and two viable persistence choices. Select one of each with a written tradeoff.
4. Define order state transitions, payment idempotency, inventory reservation, and compensation behavior.
5. Calculate rough capacity, storage growth, and data transfer. Use current prices in AWS Pricing Calculator rather than copying an old estimate.
6. Describe three failures: duplicate payment callback, unavailable database, and a failed deployment. Explain detection, customer behavior, and recovery.
7. Produce a threat model, telemetry plan, deployment plan, backup/restore plan, and cost allocation approach.
8. Review all six pillars and prioritize five improvements with owners and acceptance criteria.

**Verification:** follow one successful order, one duplicate request, and one timeout through every component. Confirm every arrow has a protocol or event contract and every data store has an owner, retention policy, and recovery approach. Ask whether the design still works when one dependency is slow rather than completely unavailable.

**Optional implementation:** deploy only an API, a queue, and a worker with synthetic order records. Prove duplicate handling and one recovery scenario. Estimate request, compute, logging, storage, and transfer costs first. Delete all temporary resources, queues, alarms, log groups, images, and schedules after retaining redacted evidence.

## Architecture decision record template

```text
Decision and date:
Problem and constraints:
Options considered:
Chosen approach and reasons:
Consequences and risks:
Evidence or experiment:
Conditions that would trigger reconsideration:
Owner:
```

Keep decisions short enough to maintain. Record uncertainty honestly, and attach measurements instead of implying that a diagram proves performance or reliability.

## Pitfalls and mastery

- Starting with a favorite service before understanding the workload.
- Using microservices or multiple Regions as a proxy for engineering quality.
- Treating managed services as eliminating application-level responsibilities.
- Omitting identity, deployment, observability, and operational ownership from diagrams.
- Claiming unlimited scale without examining partitioning, quotas, and dependencies.
- Optimizing theoretical cost while ignoring team time and business risk.

- [ ] I can turn business goals into measurable quality requirements.
- [ ] I can explain the six pillars and a real tradeoff between them.
- [ ] I can estimate traffic, concurrency, storage, and dependency fan-out.
- [ ] I can defend compute, data, and integration choices against alternatives.
- [ ] I can explain consistency and idempotency at the business-operation boundary.
- [ ] I can review failures, security, cost, and deployment behavior without relying on icons.

## Official references

- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Amazon Builders' Library](https://aws.amazon.com/builders-library/)
- [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/)
- [AWS Pricing Calculator](https://calculator.aws/)

Continue to [organizations and governance](19-organizations-and-governance.md).
