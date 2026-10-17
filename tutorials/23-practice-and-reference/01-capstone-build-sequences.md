# 01 — Build the ten portfolio projects in controlled increments

[Practice index](README.md) · [All tutorials](../README.md) · [Original project requirements](../../23-projects-and-portfolio.md)

These build sequences connect the subject tutorials into finished projects. Follow the linked technical procedures for exact service configuration, then use the acceptance checks here. Start with two projects, finish their recovery/cleanup work, and expand. A finished small project is stronger evidence than ten untested diagrams.

For every project, define inputs, expected user behavior, account/region, data size, test traffic, cost allowance, permission boundaries, and a resource ledger before building. Use synthetic data. The instructions are not a claim that these deployments have run in your account.

## 1. Private-origin static site

1. Complete [storage](../07-storage-and-backup/README.md), [networking](../05-networking-and-dns/README.md), and [IaC](../12-infrastructure-as-code/README.md) prerequisites.
2. Create a tiny HTML page and versioned asset filenames. Test relative links locally.
3. Create a dedicated private S3 origin; keep public access blocked and inspect object ownership/encryption.
4. Add CloudFront with the supported private-origin access mechanism. Use the distribution hostname first; add a custom domain/certificate only if required.
5. Configure cache behavior, HTML versus immutable asset caching, and deliberate error handling. Record the certificate location requirements for the chosen service.
6. Verify HTTPS delivery, denied direct anonymous origin access, correct missing-object behavior, and a content update under the cache policy.
7. Publish a broken asset reference, detect it, and restore the known-good version. Automate deployment of the same assets through reviewed infrastructure/pipeline configuration.

**Evidence:** origin policy, browser/network checks, cache/update results, rollback, estimate. **Cleanup:** disable/delete only the lab distribution according to its lifecycle, then dedicated objects/bucket and optional logs/DNS resources; domain registrations have separate lifecycle/cost.

## 2. Three-tier application

1. Follow [compute](../06-compute-and-load-balancing/README.md), [database](../08-databases-and-caching/README.md), and [security](../09-security-and-compliance/README.md) tutorials.
2. Run a small application locally with a health endpoint and a transactional schema. Record database connection/secret configuration without credentials in source.
3. Create a two-AZ network design and deploy an ALB plus replaceable application instances. Keep database ingress restricted to the application path.
4. Add the approved private database and migration process. Distinguish availability settings from read scaling and regional DR.
5. Verify an end-user write/read, target health, database isolation, request logs, and replacement of one application instance without loss of durable data.
6. Break one database connection setting in the sandbox; distinguish a secret error from a network timeout, then restore it.
7. Recreate from IaC and measure a data restore before declaring completion.

**Evidence:** data-path diagram, access tests, replacement/restore timings. **Cleanup:** application/network resources and database according to retention decisions; inventory final snapshots, EBS volumes, endpoints, NAT and logs.

## 3. Authenticated asynchronous job API

1. Build the [idempotent worker](../10-serverless-and-integration/01-idempotent-event-worker.md) and [API/workflow](../10-serverless-and-integration/02-api-and-workflow-integration.md) tutorials.
2. Define create/status endpoints, authenticated ownership, job states, and a stable idempotency key.
3. Deploy the minimum API, durable job store, queue, and worker with separate roles.
4. Verify valid submission, invalid input rejection, cross-user status denial, and repeated submission producing one logical result.
5. Force one synthetic job to fail. Observe retry/visibility/DLQ behavior, repair the cause, and redrive under the idempotency contract.
6. Add backlog-age/error telemetry and bound concurrency against the downstream dependency.
7. Automate build/deployment and test rollback of a bad worker version.

**Evidence:** state transitions and duplicate/failure traces. **Cleanup:** remove event mappings before dependencies where required, then dedicated queues/API/functions/table/data/logs and test identities.

## 4. Container delivery platform

1. Complete [container](../11-containers-and-kubernetes/README.md) and [delivery](../13-devops-and-delivery/README.md) tutorials.
2. Build a small non-root image, review its dependencies/content, and capture its digest.
3. Push to a dedicated ECR repository and deploy one ECS service with health checks, workload identity, logs, and a controlled network path.
4. Build once in CI, then promote the same artifact/configuration with short-lived deployment credentials.
5. Deploy an image with a failing health check; verify replacement/rollback behavior.
6. Deploy a separately controlled functional defect that passes basic health checks; prove business-level tests/telemetry catch it.
7. Compare an EKS variant only if it addresses a specific platform requirement; include cluster/network/upgrade/identity responsibilities in the decision.

**Evidence:** commit-to-digest lineage, negative pipeline checks, rollback. **Cleanup:** services/tasks/nodes, ALB, images, logs, networking, and any dedicated cluster/control plane.

## 5. Governed analytical lake

1. Complete the [batch lake tutorial](../14-data-engineering-and-analytics/01-governed-batch-lake.md).
2. Create a data contract for a synthetic sales dataset: schema, business key, event timestamp, partition rule, retention, owner, and quality checks.
3. Store immutable raw samples; transform into curated columnar output and register the supported catalog/table representation.
4. Verify counts and business aggregates, quarantine malformed rows, and rerun the batch without duplicate effects.
5. Grant a dedicated analyst only the intended data and test both permitted and denied access, including key/storage/catalog layers.
6. Compare identical queries before/after partition/file-format optimization and record scanned bytes and result equivalence.
7. Add a late file and a schema change; backfill under a documented compatibility policy.

**Evidence:** contract, lineage, quality report, authorization checks, query comparison. **Cleanup:** raw/curated/results storage, transformations, catalog resources, optional BI resources and retained versions.

## 6. Streaming telemetry

1. Use the [streaming tutorial](../14-data-engineering-and-analytics/02-streaming-and-warehouse-design.md).
2. Generate a bounded event set with unique IDs, event times, partition keys, deliberate duplicates, and late arrivals.
3. Choose one supported stream/processor path and explain why a simpler delivery service does or does not suffice.
4. Implement checkpoint/replay and destination semantics. Verify unique-event accounting and late-event policy.
5. Pause the consumer, measure lag growth, resume it, and observe catch-up within the retention window.
6. Replay into an isolated destination and compare results; do not overwrite the primary output to conceal a failed replay.
7. Estimate idle and peak service cost, including state/storage and downstream processing.

**Evidence:** event ledger, lag/replay results, ordering/deduplication contract. **Cleanup:** producers, streams/brokers, jobs/checkpoints, destinations, logs; streaming systems can remain billed while idle.

## 7. Evaluated AI assistant

Follow the complete [AI capstone](../15-ai-ml/17-ai-capstone.md): deterministic local baseline → inspected retrieval → authorized tools → evaluated generation → optional hosted runtime. Use the same fixtures at every stage.

Verify tenant isolation, source versions/deletion, citation support, abstention, prompt-injection resistance at action boundaries, bounded calls, and latency/cost. Compare a simple workflow with agent autonomy using measured task outcomes.

**Evidence:** fixture/rubric, model/prompt/index/tool versions, negative tests, trace, release decision. **Cleanup:** model/runtime/knowledge-base/vector-store/gateway/memory/evaluation/logging resources independently; preserve sanitized local evidence.

## 8. Multi-account governance

1. Begin with the [governance design tutorials](../19-organizations-and-governance/README.md); review account-plan implications before live setup.
2. Draw management, security/log archive, and workload boundaries with ownership and emergency access.
3. Draft one narrow policy control, its purpose, expected allowed/denied calls, and rollback procedure.
4. In a permitted organization sandbox only, test the control first in a staging OU/member account.
5. Verify identity permissions and organizational boundaries separately. An SCP does not grant permissions.
6. Inspect centralized evidence delivery and demonstrate who can read versus alter it.
7. Write account onboarding, exceptions with expiry, and decommission procedures.

**Evidence:** policy truth table, test results, logging access separation, exception workflow. **Cleanup:** remove only lab controls/resources after dependency review; closing accounts is a separate deliberate administrative action.

## 9. Disaster recovery

1. Complete [reliability and recovery](../17-reliability-and-disaster-recovery/README.md).
2. Define a specific failure, RTO/RPO, known synthetic records, and all application dependencies.
3. Create the chosen backup/replication path and verify permissions/key access in the recovery environment.
4. Recover to an isolated target from the written runbook.
5. Validate records, checksums/business queries, application behavior, and newest recovered data; measure elapsed time and loss.
6. Reconcile writes and rehearse failback. Explain why replication alone may copy accidental corruption/deletion.
7. Update the design/runbook when measured recovery misses the objective.

**Evidence:** timed restore report, dependency gaps, recovery/failback checks. **Cleanup:** test target and temporary copies while preserving deliberately retained backups with owner/expiry/cost.

## 10. Migration and modernization

1. Follow [migration tutorials](../21-migration-hybrid-and-edge/README.md).
2. Inventory dependencies, data, owners, licenses, downtime tolerance, and a baseline of current behavior.
3. Choose a migration strategy per component and identify compatibility changes.
4. Migrate a small synthetic dataset through full load and supported change capture where appropriate.
5. Reconcile counts plus business aggregates; test application behavior and target performance.
6. Rehearse cutover with stop-write/final-sync/validation checkpoints and a clear rollback boundary.
7. Simulate failure before cutover, then reason through failure after target writes begin; include reverse replication/reconciliation if rollback needs it.

**Evidence:** dependency map, migration validation, cutover/rollback results. **Cleanup:** temporary replication and double-running resources only after validation/retention decisions; document source decommission ownership.

## Finish and present

For each selected project, provide the problem, constraints, diagram, reproducible setup, security decisions, validation, failure/recovery evidence, cost assumptions, limitations, and exact cleanup. Use [documentation practice](04-documentation-and-maintenance.md) to prepare a readable portfolio. Clearly label unexecuted design extensions instead of presenting them as tested infrastructure.
