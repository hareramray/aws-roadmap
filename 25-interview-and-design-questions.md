# 25 — Interview questions and system-design practice

[Roadmap index](README.md) · Review after building [projects](23-projects-and-portfolio.md)

Use these questions to find gaps, not memorize scripted answers. A strong answer states assumptions, explains the mechanism, identifies a tradeoff, and describes how to verify the result. The answer notes below are checkpoints, not exhaustive model answers.

## Foundations and identity

1. **What does shared responsibility change between EC2 and Lambda?** Discuss infrastructure versus guest OS/runtime/application/data responsibilities, supported runtimes, configuration, dependencies, and access policies. Managed execution does not remove application security duties.
2. **Region, AZ, and edge location: why are they different?** Explain failure isolation, latency, service scope, and why a cache at an edge location is not a full application region.
3. **What happens when a role has an allow but an SCP denies the action?** Trace applicable policy layers. An SCP sets a boundary; it does not grant access. Include the relevant organizational scope and explicit-deny behavior.
4. **Why prefer temporary credentials?** Discuss lifetime, federation, workload roles, revocation considerations, and the risk of secret distribution. Temporary credentials still require protection.
5. **How do role trust and role permissions differ?** One determines who can assume the role; the other determines what an assumed session may do. Both can be constrained further.
6. **Why can S3 access fail even with an identity-policy allow?** Consider bucket/access-point policies, explicit denies, KMS permissions, endpoint policies, ownership/object context, organization controls, and wrong resource/region assumptions.
7. **How would you authorize a CI pipeline?** Explain federation/OIDC trust conditions, narrowly scoped deployment roles, environment separation, auditability, and protection of workflow changes.
8. **Authentication versus authorization?** Identifying a caller does not prove that caller can read another tenant's data. Describe server-side resource ownership checks.

## Networking and compute

9. **What makes a subnet public?** A route to an internet gateway is part of the answer; IPv4 internet reachability also needs suitable addressing and permitted traffic. Public subnet does not mean every resource is reachable.
10. **Why can a private instance fail to reach S3?** Inspect DNS, routes, endpoints or NAT, endpoint policy, security controls, and application credentials. Network reachability and authorization are separate.
11. **Security group versus NACL?** Stateful resource-level control versus stateless subnet-level rules; discuss response traffic, rule evaluation, and ephemeral ports.
12. **Why does a NAT gateway not accept arbitrary unsolicited inbound connections?** Explain translation state and outbound connection flow. Then distinguish IPv6 egress-only gateway behavior.
13. **ALB versus NLB?** Compare protocol layer, HTTP routing, transport behavior, static-address needs, target types, TLS configuration, and health checks.
14. **An ALB returns 502/503/504: what do you inspect?** Distinguish target availability, connection/response problems, and timeout behavior using metrics/logs; do not diagnose from status code alone.
15. **Why is a larger instance not always the solution?** Identify CPU, memory, disk, network, connection pools, locks, dependency latency, and hot keys. Measure the bottleneck first.
16. **What happens during an Auto Scaling replacement?** Cover launch configuration, bootstrapping, health/readiness, graceful draining, durable state, and scaling stabilization.
17. **Why use Spot carefully?** Explain interruption tolerance, checkpointing, diversified capacity, retryable tasks, and the distinction between low price and workload suitability.
18. **Why is peering not automatically a transit network?** Discuss explicit routing, nontransitivity, overlapping ranges, DNS, and when Transit Gateway or other architectures fit.

## Storage and databases

19. **S3, EBS, and EFS: what selects the right one?** Object API versus block device versus shared filesystem, access patterns, performance, failure scope, and operational expectations.
20. **Does S3 use eventual consistency for ordinary object reads after a write?** Explain its current strong consistency for supported object operations, then distinguish replication lag, caches, and application-level concurrency.
21. **Does versioning equal backup?** Discuss same-account compromise, accidental deletion of versions, retention, replication, immutable copies, and restore testing.
22. **RDS Multi-AZ versus read replica?** Distinguish availability and read scaling, replication behavior, failover/promotion, and deployment-type differences.
23. **When would DynamoDB be a poor choice?** Discuss unknown/ad hoc access patterns, relational joins, transactional requirements, hot keys, and the cost of secondary-index/data-model design.
24. **Why can a DynamoDB table throttle with apparently modest traffic?** Consider partition distribution, index capacity, request sizes, bursts, quotas, and the actual capacity mode.
25. **How do you prevent cache stampedes?** Cover TTL jitter, single-flight/request coalescing, stale data strategy, prewarming, and protecting the source under cache failure.
26. **What is your consistency requirement?** Define observable user behavior rather than saying “strong is always better.” Include read-your-writes, concurrent updates, transactions, and multi-region implications.
27. **What must a backup restore test prove?** Decryptability, access, integrity, application compatibility, dependencies, elapsed time, data age, and operating instructions.

## Serverless, events, containers, and delivery

28. **Why does at-least-once delivery require application work?** A duplicate may arrive after the first attempt committed. Use durable deduplication or idempotent state transitions; a queue alone does not guarantee an exactly-once business effect.
29. **SQS versus SNS versus EventBridge?** Explain buffering/work distribution, publish-subscribe notifications, and event routing/integration; consider ordering, filtering, replay, and delivery/error handling.
30. **Why can a Lambda function overwhelm a database?** Explain concurrency amplification, connection limits, pooling/proxy options, queueing, backpressure, and reserved concurrency tradeoffs.
31. **What is a poison message?** Explain repeated failure, visibility/retry configuration, DLQ/redrive, root-cause repair, and safe replay.
32. **What should an idempotency key represent?** A logical operation, not a new random value on every retry. Discuss scope, payload mismatch, persistence, expiry, concurrent requests, and in-progress states.
33. **ECS or EKS?** Choose based on application/platform requirements and operational capacity. Kubernetes familiarity is not itself a business requirement.
34. **Image tag versus digest?** A tag can move; a digest identifies image content. Discuss traceability, scanning, signing where used, and promotion of the same artifact.
35. **Why can a deployment succeed while users see failures?** Resource creation and health probes may not validate business behavior, permissions, migrations, dependencies, or geographic traffic.
36. **What makes a schema change rollback-safe?** Discuss expand/contract migrations, compatibility windows, dual reads/writes only when justified, and why rolling back code does not automatically undo data changes.
37. **What must IaC state be protected from?** Unauthorized read/write, concurrent changes, deletion, version mismatch, and secret exposure. Explain drift and why a plan deserves review.
38. **Can you rebuild from an empty environment?** Cover state/bootstrap dependencies, secrets, artifacts, DNS, permissions, quotas, and configuration outside the main template.

## Operations, security, and architecture

39. **CloudWatch, CloudTrail, and Config: which question does each answer?** Operational telemetry, API/audit activity, and resource configuration/compliance history. Coverage depends on what was enabled and collected.
40. **What is a useful alert?** User impact or a leading signal tied to action, ownership, severity, and a runbook. Avoid alerts that nobody can interpret or act on.
41. **RTO versus RPO?** Time to restore versus acceptable data loss measured in time. Both must be tested against specific failure scenarios.
42. **Why is active-active not automatically better?** Discuss conflict resolution, operational complexity, dependencies, replication, cost, and the actual business requirement.
43. **How do you investigate an unexpected bill?** Identify time/account/region/service/usage type, distinguish rate from quantity, trace deployments/traffic, contain the cause, and verify later billing updates.
44. **What is the first response to a leaked credential?** Revoke/rotate, contain unauthorized access, preserve relevant evidence, investigate activity, and address the leak path. Removing a Git line alone is insufficient.
45. **How do you design a multi-tenant service?** Tenant identity, authorization, isolation level, partitioning, noisy-neighbor controls, encryption, operational access, and per-tenant observability/cost.
46. **Why can a retry policy cause an outage?** Amplified load, synchronized retries, retry layers, and long queues. Discuss backoff with jitter, deadlines, bounded attempts, and load shedding.
47. **What does least privilege look like over time?** Start from required actions/resources, verify access, observe legitimate usage, review changes, remove unused grants, and handle exceptions with ownership/expiry.
48. **How do you choose a managed service?** Fit, limits, reliability, security, cost at expected load, skills, lifecycle, integration, and migration/exit requirements.

## Data and AI

49. **OLTP versus OLAP?** Transactional access versus analytical scans/aggregates, data layout, concurrency, latency, freshness, and why one store rarely optimizes every pattern.
50. **What makes a pipeline repeatable?** Deterministic inputs/configuration, idempotent writes, checkpoints, lineage, schema contracts, and explicit backfill semantics.
51. **Why can an analytical query be expensive?** Scanned data, partition pruning, file format/size, joins, concurrency, and provisioned resources. Compare equivalent queries, not different results.
52. **Event time versus processing time?** Explain late/out-of-order data, windows, watermark choices, and correction policies.
53. **Bedrock versus SageMaker AI?** Model access/application integration versus broader custom ML lifecycle needs; compare control, model choices, deployment, evaluation, and economics for the actual use case.
54. **Why does RAG still need evaluation?** Retrieval can miss or expose wrong data, context can conflict, and generation can invent answers. Measure retrieval quality, groundedness, access isolation, latency, and cost.
55. **How would you handle prompt injection?** Treat untrusted text as data, constrain tools, authorize outside the model, separate privileges, test attacks, and limit impact. Prompt wording alone is not a security boundary.
56. **How do you measure an AI feature's value?** Task success on representative cases, failure severity, human review needs, latency, cost per task, and comparison with a simpler baseline.

## System-design exercises

Use the same sequence for each: clarify users and requirements; estimate load/data; define APIs and data model; draw the request path; identify trust boundaries; choose scaling and failure behavior; define telemetry/recovery; estimate costs; discuss alternatives.

### A. Appointment booking

Prevent double booking while allowing browsing under heavy traffic. Explain transaction/conditional-write choices, time zones, idempotent booking requests, asynchronous notifications, and behavior after a timeout. A cache may accelerate availability views but cannot independently guarantee booking correctness.

### B. Image processing pipeline

Accept uploads, generate variants, track status, and limit abusive content/requests. Discuss private uploads, presigned URL scope/expiry, event duplicates, job ownership, DLQs, object lifecycle, cache behavior, and preventing unbounded compute work.

### C. Multi-region customer API

Clarify whether the requirement is low latency, regional disaster recovery, or both. Define authoritative writes, replication lag, conflict policy, routing/failover, dependencies, key availability, and recovery testing. Compare a simpler single-region multi-AZ design.

### D. Enterprise analytics platform

Combine several sources with different schemas and access rules. Explain contracts, ingestion modes, catalog/lineage, table formats, data quality, row/column restrictions, deletion, backfill, and cost attribution. Avoid treating a bucket of files as a governed data platform by itself.

### E. Internal knowledge assistant

Users must only retrieve documents they may already access. Define identity propagation, document permissions, ingestion updates/deletes, retrieval authorization, evaluation, citations, unanswerable responses, tool restrictions, and incident handling for incorrect or exposed content.

## Behavioral evidence

Prepare examples of an incident you investigated, a design you simplified, a tradeoff you documented, a mistake you corrected, and a topic you learned from feedback. State your personal contribution and measured outcome. For a learning project, say it was a lab; do not represent it as commercial production experience.

Use [the troubleshooting playbook](31-troubleshooting-playbook.md) to deepen weak diagnostic answers and [the templates](32-engineering-templates.md) to structure your project stories.
