# 23 — Projects and portfolio

[Roadmap index](README.md) · Use after the relevant technical chapters

A portfolio should show how you think, build, verify, and operate. Complete two or three projects thoroughly before starting all ten. Use synthetic data and an authorized sandbox. Every deployment needs a current estimate, resource inventory, bounded test workload, and cleanup plan. These are project specifications, not pretested deployment packages.

## Common project contract

Each project should contain a README, architecture diagram, infrastructure definitions, application code where applicable, sanitized example configuration, tests, deployment instructions, operating runbook, cost model, and known limitations. Explain what you chose and what you rejected. Include evidence of a failure and recovery, not just the successful console screen.

Set a measurable requirement before implementation: expected traffic, data size, latency target, availability target, recovery objectives, maximum experiment duration, and budget. Choose educational values rather than inventing production claims. A short lab test does not demonstrate an annual availability percentage.

## Project 1: private-origin static site

**Learn first:** 04, 05, 07, 09, 12.  
**Architecture:** browser → CloudFront → private S3 origin; optional Route 53 and ACM for a custom domain.

Build a static documentation site with a private S3 REST origin and CloudFront Origin Access Control. Keep S3 public access blocked. If using a custom CloudFront domain, follow the current certificate-region requirements. A custom domain is optional and can add registration/DNS costs.

Implement cache-control headers for immutable versioned assets and an appropriate policy for HTML. Configure a deliberate error-page or SPA fallback strategy; do not blindly convert every missing resource to HTTP 200. Deploy assets through a repeatable script or pipeline. Record why a public S3 website endpoint has different origin/security characteristics.

**Acceptance criteria:** the CloudFront URL serves the page over HTTPS; direct anonymous S3 object access fails; a content update behaves according to your cache policy; missing assets produce intended status codes; no credentials appear in built assets. Explain the difference between cache invalidation and versioned filenames.

**Failure drill:** publish a bad asset reference, identify it through browser/network evidence, and roll back to a known version. Do not remove access controls to “fix” an origin-policy mistake.

**Cost/cleanup:** inventory distribution, bucket contents and versions, optional DNS zone/domain, logs, and certificates. Disable/delete the distribution through the documented process before removing its origin dependencies. Domain registrations may remain billed independently of site teardown.

## Project 2: resilient three-tier application

**Learn first:** 04–09, 12, 16–18.  
**Architecture:** ALB → application instances across two AZs → private relational database.

Use a launch template, Auto Scaling group, application health endpoint, restricted security groups, and a secret-retrieval mechanism. Choose instance access through Systems Manager where suitable. Explain the network path required for management and package downloads; private instances do not automatically have internet egress.

Model an order transaction with a schema migration and parameterized queries. Start with an intentionally small database; evaluate Multi-AZ as a reliability design and deploy it only if your lab budget supports it. Distinguish a Multi-AZ DB instance standby from a read replica and from other RDS cluster topologies.

**Acceptance criteria:** the database is not publicly reachable; application instances accept inbound application traffic only from the ALB security group; replacing one instance does not lose durable data; logs identify request failures; the application can recover connections after a database interruption. Record measured recovery time rather than asserting zero downtime.

**Failure drill:** terminate one disposable application instance and observe health checks, capacity replacement, and user impact. Separately simulate an invalid secret or blocked database path and distinguish their symptoms.

**Cost/cleanup:** database runtime/storage/backups, ALB, EC2/EBS, public addresses, NAT/endpoints, logs. Decide which final snapshot to retain, who owns it, and when to remove it. Stopping compute does not remove all storage or networking charges.

## Project 3: authenticated serverless job API

**Learn first:** 04, 08–10, 12–13, 16.  
**Architecture:** API Gateway → Lambda → DynamoDB; SQS → worker Lambda for asynchronous jobs; Cognito or another supported identity provider.

Provide job creation and status endpoints. Validate request schemas and authorization on the server. Use a stable idempotency key for job creation, conditional writes for state transitions, and a bounded queue retry strategy. Record states such as `accepted`, `running`, `succeeded`, and `failed` with timestamps.

Give the producer and worker different execution roles. Configure a DLQ or documented redrive path and monitor backlog age. Apply an explicit concurrency policy so a burst cannot overwhelm downstream services. Explain why a successful API response can mean “accepted for processing,” not “completed.”

**Acceptance criteria:** unauthenticated or cross-user access is rejected; repeated identical job submissions do not duplicate the business result; an invalid message reaches the failure path; a worker restart does not lose accepted work; traces/logs correlate API request and job ID.

**Failure drill:** force the worker to fail for one synthetic job, observe retry/visibility behavior, fix the cause, and redrive safely. Prove that duplicate delivery does not create duplicate results.

**Cost/cleanup:** requests, duration, queue usage, database requests/storage, identity features, traces/logs. Delete test users and test data; remove event-source mappings before deleting dependencies where necessary.

## Project 4: container delivery platform

**Learn first:** 05–06, 09, 11–13, 16.  
**Architecture:** source repository → build pipeline → ECR → ECS service on Fargate or EC2 → ALB.

Package a small API, pin dependencies, build once, scan the image, and deploy by digest. Use workload roles, a separate deployment identity, and short-lived CI credentials. Include readiness/health behavior, graceful shutdown, structured logs, secrets injection, and resource requests/limits appropriate to the platform.

Use infrastructure code for the registry, cluster, service, task definition, networking, and permissions. Add a deployment strategy with a clear rollback condition. Record the difference between build failure, deployment failure, and a healthy deployment with a faulty business response.

**Acceptance criteria:** a commit produces a traceable image; a failed application check blocks release; a broken image fails deployment or triggers rollback; the application cannot read unrelated secrets; restarting tasks does not erase externalized state.

**Failure drill:** deploy an image that fails health checks, then a separately controlled image with a functional defect. Show which failure your pipeline detects and which requires application-level telemetry.

**Extension:** implement the same workload on EKS only after explaining the operational benefit. Compare control-plane cost, upgrades, scheduling, workload identity, and network address consumption.

**Cost/cleanup:** running tasks/nodes, load balancers, registry storage, builds, endpoints/NAT, logs, optional EKS control plane. Remove untagged images and retained logs according to a deliberate policy.

## Project 5: governed analytical data lake

**Learn first:** 04, 07–09, 12, 14, 16.  
**Architecture:** synthetic source data → S3 raw zone → Glue transformation/catalog → curated tables → Athena → optional BI.

Create a source data contract with schema, partition field, timestamp meaning, primary business key, and data-quality expectations. Ingest daily files, retain lineage, quarantine invalid rows, and produce partitioned columnar output. Compare query scans before and after partitioning/columnar conversion using the same logical query.

Apply access rules to raw and curated data separately. If using Lake Formation, explain its interaction with IAM, S3, and encryption-key permissions. Include a schema evolution example and a repeatable backfill. Avoid tiny-file proliferation and overly fine partitions.

**Acceptance criteria:** malformed data is visible and does not silently corrupt curated results; rerunning a batch does not double-count; a least-privilege analyst can query approved data; an unauthorized principal cannot; counts and aggregates reconcile with the source.

**Failure drill:** add a late file and a schema-incompatible file. Demonstrate detection, quarantine or controlled evolution, and corrected downstream results.

**Cost/cleanup:** transformation compute, scanned bytes, catalog usage, output storage, optional BI subscriptions. Delete query result objects and historical test versions as well as input data. Document any legal/retention assumptions as exercise assumptions.

## Project 6: streaming telemetry pipeline

**Learn first:** 07–10, 14, 16–17.  
**Architecture:** bounded synthetic producers → Kinesis Data Streams or MSK → stream processor → durable storage and operational metrics.

Choose an event key and explain ordering scope, hot partitions, retention, consumer lag, and replay. Process event time separately from ingestion time. Include duplicates, out-of-order events, and an explicit watermark/window strategy if performing aggregations.

Choose one streaming platform based on requirements; do not deploy both merely to enlarge the diagram. Compare a managed delivery service when only buffering and destination delivery are required. Keep producer rate and test duration bounded and logged.

**Acceptance criteria:** you can account for all unique test event IDs; duplicate effects are controlled; late events follow a documented policy; a stopped consumer can catch up within the retention window; lag and rejected records are observable.

**Failure drill:** pause the consumer, restart it, and measure backlog recovery. Replay a known interval into an isolated destination to prove reprocessing without corrupting the primary output.

**Cost/cleanup:** stream capacity, broker/control-plane resources where relevant, processing runtime, delivery requests, retention, and downstream storage. Streaming services can incur substantial idle charges; use a design-only variant if the sandbox budget is small.

## Project 7: evaluated retrieval-augmented assistant

**Learn first:** 04, 07–10, 15–16, 20.  
**Architecture:** approved document set → chunking/embeddings → retrieval store → Bedrock model → authenticated application.

Use synthetic or public documents. Record document versions, chunking strategy, embedding model/version, retrieval parameters, and access metadata. Create an evaluation set before tuning: answerable questions, unanswerable questions, conflicting passages, and malicious instructions embedded in retrieved text.

Require citations to retrieved material where applicable, an explicit insufficient-evidence response, and server-side authorization for retrieval. Treat retrieved text as untrusted data. If adding tools, enforce a narrow action schema and authorization independently of the model's output. Set request/token budgets and monitor latency and cost per successful task.

**Acceptance criteria:** evaluations report groundedness, retrieval relevance, refusal/abstention behavior, and unauthorized-document exposure tests; the app does not execute instructions found in documents; deleting a document updates the retrieval index; quality improvements are measured on held-out cases.

**Failure drill:** inject a misleading document and an access-control mismatch. Demonstrate both content-quality and authorization controls. A guardrail feature alone is not evidence that the application is secure.

**Cost/cleanup:** model input/output tokens, embeddings, retrieval-store runtime/storage, ingestion jobs, traces. Remove vector indexes, documents, endpoints, and model-related artifacts according to retention decisions.

## Project 8: multi-account governance design

**Learn first:** 04, 09, 12, 16, 19–20.  
**Architecture:** management, security/log archive, and workload accounts with OUs and identity assignments.

Begin as a design exercise. Organization/Control Tower changes can affect Free Tier eligibility; review [account setup](02-account-and-tooling.md) before creating anything. In an approved multi-account sandbox, implement a minimal landing-zone slice with central audit collection, account vending assumptions, baseline controls, and an exception process.

Define which controls are preventive, detective, or proactive. Explain why an SCP limits permissions but does not grant them. Test a harmless denied action in a disposable member account. Keep emergency access and recovery paths documented, and do not apply untested deny policies to a real management structure.

**Acceptance criteria:** account boundaries follow ownership and risk; log access is separated from workload administration; policy effects are verified; exceptions have owner/expiry; a new workload account has a documented onboarding procedure.

**Failure drill:** diagnose a denied action caused by an SCP versus an identity policy. Show how an approved policy change would be tested and rolled back.

**Cost/cleanup:** audit/data event volume, security services, Config evaluations, log storage, cross-account delivery, account lifecycle obligations. Do not treat closing accounts as a routine automatic lab cleanup step.

## Project 9: measured disaster recovery

**Learn first:** 07–09, 16–18.  
**Architecture:** primary application/data plus a chosen backup-and-restore, pilot-light, or warm-standby recovery approach.

Choose explicit RTO/RPO targets and a failure scenario. Include infrastructure, data, secrets, certificates, DNS, identity, quotas, and operational communication. A database backup without application configuration is not a complete recovery plan.

Create known synthetic records with timestamps, take backups or establish replication, and recover into an isolated target. Check data integrity and application behavior before shifting any test traffic. Include failback and reconciliation for writes made during recovery.

**Acceptance criteria:** recovery works from written instructions; measured recovery time and data loss are reported; a missing dependency is identified and repaired; restore permissions and encryption-key access are verified; cleanup does not remove the only retained backup.

**Failure drill:** make the primary test application unavailable or simulate loss of one dependency. Keep the drill bounded to your sandbox. Compare observed results with the stated objectives and revise the architecture if they do not fit.

**Cost/cleanup:** duplicate regional resources, replication, cross-region transfer, backup retention, idle standby capacity. Document what persists intentionally and its monthly cost estimate.

## Project 10: migration and modernization case study

**Learn first:** 05–09, 12–13, 17–21.  
**Architecture:** local or isolated source application/database → assessed target architecture → staged migration and cutover.

Inventory dependencies, owners, data sensitivity, licensing constraints, downtime tolerance, and rollback criteria. Choose a migration strategy per component. Rehosting a server and refactoring its database are different workstreams with different risks.

Use a small synthetic database for a full load plus change replication where supported. Validate schema compatibility, row counts, checksums or business aggregates, and application queries. Build a cutover runbook with stop-write, final-sync, validation, DNS/connection changes, and rollback checkpoints. Define the point after which rollback requires reverse replication or reconciliation.

**Acceptance criteria:** dependencies are documented; target performance meets a defined baseline; migration checks detect intentionally altered data; cutover and rollback are rehearsed; source decommissioning happens only after agreed validation and retention criteria.

**Failure drill:** fail validation before cutover and abort safely. Separately reason through a failure after target writes begin; explain why simply switching DNS back may lose or diverge data.

**Cost/cleanup:** replication infrastructure, temporary double-running, storage/transfer, backup retention, licenses. Avoid leaving source and target billed indefinitely because the technical cutover succeeded but ownership was unclear.

## Portfolio assessment

Score each dimension 0–3: absent, described, demonstrated, and repeatably demonstrated. Evaluate requirements, architecture, security, infrastructure, application correctness, delivery, observability, resilience, cost, and documentation. This is a self-assessment rubric, not an external certification standard.

A strong project explains an important limitation honestly. “Tested with 500 synthetic events and one worker failure” is better evidence than “enterprise-grade and infinitely scalable.” Publish sanitized artifacts, omit account identifiers and secrets as appropriate, and make it possible for a reviewer to understand the project without paying to deploy it.

Use [engineering templates](32-engineering-templates.md), [the completion checklist](26-master-completion-checklist.md), and the official documentation linked by each technical chapter when implementing these briefs.
