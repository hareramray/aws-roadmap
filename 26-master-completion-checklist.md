# 26 — Master completion checklist

[Roadmap index](README.md)

Mark a box only when you can point to evidence: a diagram, explanation, configuration, test result, failure analysis, or restore report. “Watched a tutorial” is a study activity, not proof of proficiency. For specialist tracks, choose the relevant items; explicitly mark others `surveyed` rather than pretending to have mastered every service.

## Gate 1: prerequisites and a controlled sandbox

- [ ] Explain DNS, IP routing, ports, TLS, HTTP, and a database call as one request path.
- [ ] Inspect a process, listening port, disk usage, and application logs locally.
- [ ] Write a small script with input validation, exception handling, and a useful exit code.
- [ ] Use Git branches and review diffs without committing credentials or sensitive state.
- [ ] Explain a SQL transaction, an index, and a key-value access pattern.
- [ ] Identify your AWS account, region, principal, and credential source.
- [ ] Protect root access and use an everyday non-root identity.
- [ ] Know your account plan, credit eligibility/expiry, and service restrictions.
- [ ] Configure and verify cost notifications; explain why alerts are not hard caps.
- [ ] Maintain a resource ledger and estimate a lab before deploying.

**Evidence:** local app, identity check stored privately, setup checklist, and a completed low-scope lab with cleanup. See chapters 01–02 and 33.

## Gate 2: cloud, identity, and networking

- [ ] Explain regions, AZs, global services, edge delivery, and failure boundaries.
- [ ] Map shared responsibility for EC2, a managed database, and Lambda.
- [ ] Distinguish identity policies, resource policies, trust policies, boundaries, session policies, SCPs, and RCPs.
- [ ] Diagnose an explicit deny and a missing allow without widening all permissions.
- [ ] Use a role for a workload and explain credential delivery/refresh.
- [ ] Draw public/private subnet routes for IPv4 and explain an IPv6 egress design.
- [ ] Compare security groups and NACLs with request and response paths.
- [ ] Explain NAT, gateway/interface endpoints, PrivateLink, and their cost drivers.
- [ ] Diagnose public/private DNS resolution and a wrong routing decision.
- [ ] Compare peering, Transit Gateway, VPN, and Direct Connect at a design level.

**Evidence:** network diagram, least-privilege example, and one intentionally denied request explained from policy/network evidence. See chapters 03–05.

## Gate 3: compute, storage, and data

- [ ] Select EC2 instance characteristics from measured workload needs.
- [ ] Explain launch templates, Auto Scaling, load-balancer health, and graceful shutdown.
- [ ] Separate ephemeral application state from durable state.
- [ ] Compare S3, EBS, EFS, FSx, and backup options.
- [ ] Configure private object access and explain encryption/key permissions.
- [ ] Explain S3 versioning, lifecycle, replication, and retention as different controls.
- [ ] Restore test data and validate its integrity.
- [ ] Compare relational, key-value, document, graph, time-series, search, and cache needs.
- [ ] Distinguish RDS deployment types, read replicas, and recovery choices.
- [ ] Design DynamoDB keys from access patterns and identify hot-key risks.
- [ ] Explain consistency, conditional writes, transactions, and relevant replication constraints.
- [ ] Design cache expiry, invalidation, and behavior when the cache fails.

**Evidence:** workload deployment, data model, storage decision record, and restore report. See chapters 06–08.

## Gate 4: security and application integration

- [ ] Create a threat model with assets, entry points, trust boundaries, and mitigations.
- [ ] Distinguish KMS, Secrets Manager, ACM, and application authorization responsibilities.
- [ ] Review attack surface and remove unnecessary public exposure.
- [ ] Explain detection and investigation using audit, vulnerability, and threat findings.
- [ ] Build a validated API with server-side resource authorization.
- [ ] Compare SQS, SNS, EventBridge, and Step Functions.
- [ ] Handle duplicates with durable idempotency or safe state transitions.
- [ ] Demonstrate retries, DLQ handling, and controlled replay.
- [ ] Bound concurrency and protect a slower downstream dependency.
- [ ] Trace a request across synchronous and asynchronous components.

**Evidence:** threat model, access tests, job API, and poison-message recovery. See chapters 09–10.

## Gate 5: containers, infrastructure, and delivery

- [ ] Build a minimal container image, pin dependencies, and distinguish tags from digests.
- [ ] Explain ECS task/execution roles or the equivalent EKS workload-identity arrangement.
- [ ] Configure health behavior, resource needs, logs, and graceful termination.
- [ ] Explain Kubernetes control plane, nodes, Pods, Deployments, Services, ingress, and storage.
- [ ] Know why EKS networking capacity and upgrades require planning.
- [ ] Recreate an environment with reviewed infrastructure code.
- [ ] Protect state, handle locking/concurrent changes, and inspect drift.
- [ ] Separate bootstrap dependencies from ordinary application infrastructure.
- [ ] Build once and promote a traceable artifact through environments.
- [ ] Use short-lived CI credentials with constrained trust.
- [ ] Block a broken change with relevant tests and demonstrate rollback.
- [ ] Explain how schema compatibility affects deployment and rollback.

**Evidence:** repository, pipeline result, infrastructure plan/change set, and failed-release recovery. See chapters 11–13.

## Gate 6: data engineering and AI awareness

- [ ] Explain batch versus stream, OLTP versus OLAP, and event time versus processing time.
- [ ] Define a data contract, lineage, schema evolution, and a backfill procedure.
- [ ] Measure effects of partitioning, columnar formats, and file sizes on query behavior.
- [ ] Reconcile input/output data and quarantine invalid records.
- [ ] Explain governance across catalog, object storage, keys, and analytical tools.
- [ ] Handle duplicates, late events, lag, checkpoints, and replay in a stream design.
- [ ] Distinguish custom ML training from consuming a managed foundation model.
- [ ] Explain training/validation/test splits, leakage, reproducibility, and model drift.
- [ ] Evaluate a RAG application for retrieval quality, groundedness, and access isolation.
- [ ] Treat model input/output and retrieved instructions as untrusted at tool boundaries.
- [ ] Estimate AI inference cost per task and define latency/quality targets.

**Evidence:** data pipeline or evaluated AI project, plus an explanation of the track you did not implement. See chapters 14–15.

## Gate 7: operations and reliability

- [ ] Distinguish metrics, logs, traces, audit records, and configuration history.
- [ ] Create a dashboard tied to user behavior and dependency health.
- [ ] Define an SLI, SLO, alert action, owner, and runbook.
- [ ] Investigate a failure with timestamps, request IDs, recent changes, and hypotheses.
- [ ] Operate patch/configuration processes without undocumented manual drift.
- [ ] Define RTO/RPO for a specific failure and test them.
- [ ] Recover data plus configuration, identity, secrets, and other required dependencies.
- [ ] Explain multi-AZ versus multi-region reliability and their common failure paths.
- [ ] Design bounded retries, timeout budgets, backpressure, and load shedding.
- [ ] Write an incident report that separates trigger, causes, impact, and corrective actions.

**Evidence:** incident exercise, telemetry, restore timings, and improved runbook. See chapters 16–17 and 31–32.

## Gate 8: architecture, governance, and economics

- [ ] Turn a vague requirement into measurable workload constraints.
- [ ] Review a design against all six Well-Architected pillars.
- [ ] Compare alternatives and record a decision with consequences and a review trigger.
- [ ] Define account boundaries, OU structure, ownership, and an exception process.
- [ ] Explain central audit/security administration and emergency access.
- [ ] Test organization controls safely and understand their scope/exceptions.
- [ ] Build a cost model with idle, expected, and peak scenarios.
- [ ] Explain data-transfer, request, storage, runtime, and observability cost drivers.
- [ ] Use workload measurements before buying commitments or increasing capacity.
- [ ] Allocate costs to a workload or unit of useful business output.

**Evidence:** architecture review, account design, cost worksheet, and measured optimization. See chapters 18–20.

## Gate 9: migration and specialization

- [ ] Assess application dependencies, data, licensing, owners, and business constraints.
- [ ] Choose a migration strategy per component and justify it.
- [ ] Rehearse cutover, validation, rollback, and source decommissioning.
- [ ] Explain hybrid DNS/routing, redundant connectivity, and offline/edge requirements.
- [ ] Survey IoT, media, end-user computing, HPC, contact-center, and industry use cases.
- [ ] Select one specialization and complete a deeper exercise.
- [ ] Check service lifecycle and regional/account availability before adopting a niche service.

**Evidence:** migration plan or specialist project, with dependencies and failure handling. See chapters 21–22 and 30.

## Final portfolio gate

- [ ] Two or three projects solve clear problems and can be understood without live deployment.
- [ ] Each has reproducible setup, relevant tests, security reasoning, cost assumptions, and cleanup.
- [ ] At least one includes failure diagnosis, rollback, and data recovery evidence.
- [ ] Every published claim matches the actual test scale and environment.
- [ ] Secrets and sensitive identifiers have been removed from shareable artifacts.
- [ ] I can answer the core questions in chapter 25 with mechanisms and examples.
- [ ] I know my current gaps and have a plan to revisit them.

Do not wait to know every specialist service before applying for a suitable role or building a useful system. Keep the foundation broad, deepen the skills your role requires, and label your evidence honestly.
