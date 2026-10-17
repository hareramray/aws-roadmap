# 17. Reliability and disaster recovery

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete [networking](05-networking-and-dns.md), [storage and backup](07-storage-and-backup.md), [databases](08-databases-and-caching.md), [infrastructure as code](12-infrastructure-as-code.md), and [observability](16-observability-and-operations.md).

Your goal is to build a workload that tolerates expected failures and recovers from larger disruptions with measured data loss and recovery time. You should be able to select an appropriate recovery strategy, test a restore, explain replication limitations, and execute a documented failover and failback plan.

## 1. Define what must survive

Begin with business operations and failure scenarios, not the phrase “highly available.” An internal monthly report and a live payment service have different consequences when unavailable. Identify owners, critical periods, dependencies, legal retention constraints, and manual workarounds.

Define:

- **Availability:** whether the service can complete the required operation during its measurement window.
- **Durability:** the likelihood that stored data remains intact; it does not imply immediate accessibility.
- **RTO, recovery time objective:** the maximum acceptable disruption before a usable service is restored.
- **RPO, recovery point objective:** the maximum acceptable age of missing or unrecoverable data after recovery.
- **Recovery scope:** component, Availability Zone, Region, account compromise, accidental deletion, data corruption, or external dependency failure.

Measure RTO from the business disruption, including detection, decision, provisioning, data recovery, routing, and validation. Measure RPO using committed business records, not just a dashboard's replication-lag number. A restore that starts quickly but takes hours to validate may miss the business target.

Example exercise targets might be a two-hour RTO and a fifteen-minute RPO for a small order service. These are design assumptions requiring evidence; no service selection automatically guarantees them.

## 2. Build resilience before regional recovery

Eliminate single-instance dependencies. Use appropriate multi-AZ services, health checks, load balancing, autoscaling, and replacement from immutable artifacts. Confirm which components are actually distributed: two application instances do not help when both depend on one self-managed database.

Design capacity for a failure. If a surviving Availability Zone cannot handle the load, nominal redundancy becomes an outage. Model quota headroom, database connections, queue consumers, IP addresses, EBS throughput, and external API limits. Test degradation under realistic load rather than assuming autoscaling will always catch up immediately.

Make applications tolerant of transient faults:

1. Set timeouts for every remote operation, including connection establishment.
2. Retry only suitable errors with bounded attempts, exponential backoff, and jitter.
3. Use idempotency tokens and durable deduplication when an operation may execute more than once.
4. Apply backpressure and admission control instead of accepting unlimited work.
5. Separate failure domains with bulkheads and independent capacity pools.
6. Use circuit breakers and graceful degradation where they fit the dependency contract.
7. Limit retry layers so one customer request cannot create a retry storm.

A timeout is ambiguous: the downstream operation may have succeeded before its response was lost. Retry-safe behavior requires an application decision, not just a bigger retry count. A cache can help absorb read load, but a cold cache after recovery can overwhelm the database.

## 3. Match a disaster recovery strategy to requirements

| Strategy | What exists before the event | Main tradeoff |
| --- | --- | --- |
| Backup and restore | Recoverable data plus deployable infrastructure and artifacts | Lower standing capacity; recovery needs restore and deployment time |
| Pilot light | Essential replicated state and minimal core services | Additional components must be launched or scaled before serving traffic |
| Warm standby | A functional but smaller deployment | More standing cost; still needs scaling and full dependency validation |
| Multi-site active/active | Multiple locations actively serve customers | Complex data coordination, routing, testing, and operations |

Hot standby is another active/passive arrangement with substantially ready capacity. These labels describe patterns, not guaranteed time ranges. The AWS DR guidance explains the patterns and recommends reducing failover dependence on control-plane operations where practical. [AWS disaster recovery options](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html).

Choose the least complex strategy that demonstrably meets the requirement. Multi-Region infrastructure creates additional deployment, data, identity, and operational failure modes. Some workloads need regional recovery; others gain more from tested backups, sound multi-AZ design, and simpler operations.

## 4. Make backups usable and protected

Inventory authoritative state: relational data, object versions, file systems, secrets, encryption-key dependencies, identity configuration, container images, certificates, infrastructure templates, and third-party configuration. A database snapshot alone may not reconstruct a working system.

Understand application-consistent versus crash-consistent backups. Coordinate writes where necessary and validate transaction boundaries. Select frequency and retention based on data-loss tolerance, discovery time for corruption, audit requirements, and restore expense. Point-in-time recovery helps recover before a damaging change, but only within the configured retention window.

Study AWS Backup plans, selections, vaults, recovery points, copy jobs, restore jobs, reporting, and supported resource combinations. Verify cross-account, cross-Region, continuous-backup, and encryption support for the exact resource. Support is not identical across services.

Separate backup administration from application administration. Consider protected backup copies, restrictive deletion permissions, and recovery access that survives a workload-account compromise. Learn Vault Lock governance and compliance behavior before applying it: immutable retention can deliberately prevent deletion and retain charges. Study logically air-gapped vaults and their supported recovery and sharing workflows; “air-gapped” in the product name should not be interpreted as an offline tape disconnected from every service. [AWS Backup documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html).

Do not enable irreversible compliance retention in a beginner lab. A backup success event is evidence of a completed job, not evidence that the business service can be restored. Periodic restore tests must validate data, encryption permissions, application startup, and a representative business operation.

## 5. Understand multi-Region data semantics

Replication and backup solve different problems. Replication can propagate accidental deletion, malicious writes, and application corruption. Backups preserve earlier recovery points, while replication primarily reduces the delay in maintaining another copy.

Study each data technology separately:

- **RDS and Aurora:** distinguish Multi-AZ availability from cross-Region replication and engine-specific promotion/failover behavior. Read replicas are not automatically equivalent to a full DR plan.
- **Aurora Global Database:** learn writer placement, replication behavior, planned switchover versus unplanned failover, and application reconnection.
- **DynamoDB global tables:** understand the configured consistency model, supported Regions and operations, concurrent writes, and how conflicts or strong-consistency constraints affect application design. Do not assume every global-table configuration behaves identically.
- **S3 replication:** understand eligibility, versioning, existing-object handling, delete-marker behavior, encryption permissions, and destination access.
- **Caches and queues:** decide whether state can be rebuilt, replayed, or must be independently protected. Replaying messages requires idempotent consumers.

For each store, document who may write before, during, and after failover. Prevent split brain with a clear writer authority or a deliberately designed multi-writer model. Reconciliation must preserve business invariants such as “one accepted payment per order,” even when network connectivity is partitioned.

## 6. Recover the whole dependency chain

Pre-position or reproduce images, IaC, secrets, parameter values, KMS permissions, DNS, certificates, quotas, monitoring, and operator access. Consider regional service availability, instance capacity, licensing, and outbound access. A recovery environment that depends on an unavailable primary-region artifact registry is incomplete.

Understand Route 53 routing and health checks, Global Accelerator, and Amazon Application Recovery Controller capabilities where relevant. DNS caching and client connection reuse mean a routing change does not move every client instantly. Health checks should represent the ability to serve the intended workload and avoid circular dependencies.

Write both **failover** and **failback** procedures. Failback includes reconciling data, establishing the intended writer, rebuilding replication, gradually moving traffic, and verifying correctness. Reverting a DNS record without reconciling writes can cause a second incident.

AWS Elastic Disaster Recovery can support recovery of supported server-based workloads; its replication infrastructure, target configuration, drills, and post-recovery application validation still need operational ownership. It does not remove the need to understand database consistency and dependencies.

## Practical lab: prove a restore

**Scope:** use a small disposable data store from an earlier lab. Prefer a manual restore exercise in one Region first; use a design-only extension for multi-account or multi-Region recovery if spending is constrained.

1. Create a small synthetic dataset with record counts, deterministic checksums, and a known latest transaction ID.
2. Write an IaC deployment procedure and a recovery runbook before taking the backup.
3. Create a supported backup or recovery point. Record creation time and the recoverable state.
4. Add a few known records after that point. Simulate loss only in a disposable copy, or restore into a new isolated resource without deleting the source.
5. Restore to an isolated destination. Recreate application permissions and configuration using the runbook.
6. Verify counts, checksums, referential integrity, and one complete business transaction.
7. Measure elapsed recovery time and compare the recovered transaction ID with the source to calculate observed data loss.
8. Document which steps required control-plane access, human decisions, or unavailable information.

**Verification:** another person should be able to follow the instructions using only the documented artifacts. Record an actual RTO observation, the achieved RPO for the scenario, and at least one gap. If you only perform a tabletop exercise, label timings as estimates rather than measured recovery performance.

**Cost and cleanup:** estimate snapshot storage, restored storage, temporary instances, backup copies, replication, data transfer, and encryption requests. Delete restored resources, lab snapshots, schedules, test traffic, and temporary permissions after collecting evidence. Keep production retention obligations separate from lab cleanup; immutable recovery points cannot always be removed immediately.

## Game days and advanced practice

Progress from tabletop review to a single dependency fault, an instance failure, an AZ capacity scenario, and a regional exercise. AWS Fault Injection Service can orchestrate supported experiments; define steady state, explicit scope, stop conditions, and recovery before execution. Never equate an unplanned production outage with a controlled experiment.

Include non-infrastructure failures: expired certificates, credential revocation, corrupt deployment, accidental data deletion, unavailable identity provider, lost operator access, and compromised backup credentials. Test communication and decision authority as well as technical automation.

## Pitfalls and mastery

- Replication without historical recovery points cannot reliably recover from logical corruption.
- Multi-AZ, Multi-Region, and cross-account controls protect against different failure classes.
- A nominal recovery target is not evidence; restore drills provide evidence.
- Recovery capacity, quotas, keys, permissions, and artifacts are part of the recovery system.
- Aggressive retries can turn a partial failure into a fleet-wide incident.

- [ ] I can explain availability, durability, RTO, and RPO with different examples.
- [ ] I can justify a DR strategy from business requirements and operating cost.
- [ ] I have restored data and validated a business operation.
- [ ] I can explain data loss, split brain, failover, and failback for my selected store.
- [ ] I have identified dependencies that could block recovery.
- [ ] I can run a scoped experiment with verification and stop conditions.

## Official references

- [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [AWS Backup documentation](https://docs.aws.amazon.com/aws-backup/)
- [AWS Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/)
- [AWS Fault Injection Service](https://docs.aws.amazon.com/fis/)
- [Amazon Application Recovery Controller](https://docs.aws.amazon.com/r53recovery/)

Continue to [architecture and system design](18-architecture-and-system-design.md).
