# 31 — Troubleshooting playbook

[Roadmap index](README.md) · Pair with [CLI/SDK tools](29-cli-and-sdk-reference.md) and [runbook templates](32-engineering-templates.md)

Troubleshooting is hypothesis testing. Identify the failing boundary and collect evidence before changing configuration. Use only authorized systems; keep experiments reversible and bounded. This playbook gives diagnostic order, not permission to alter production resources.

## A repeatable investigation loop

1. **Define the symptom:** which user operation fails, from where, since when, and how often?
2. **Determine scope:** account, region, environment, tenants, AZs, endpoints, data partitions, and deployment versions.
3. **Preserve evidence:** request IDs, UTC timestamps, relevant logs/metrics/traces, error codes, and recent changes. Redact secrets.
4. **State a hypothesis:** “the target cannot open a database connection” is testable; “AWS is broken” is not.
5. **Check the cheapest discriminator:** DNS result, route, permission decision, target health, dependency metric, or a minimal API call.
6. **Change one cause at a time:** prefer a rollback or narrow mitigation with a known reversal.
7. **Verify user recovery and data integrity:** a green resource status is not enough.
8. **Record the cause and follow-up:** update the runbook and add a useful preventive or detecting control.

Check AWS Health when relevant, but do not stop local diagnosis merely because an unrelated regional incident exists.

## 1. Authentication or AccessDenied

First run an identity check with the exact profile/region used by the failed action. Verify session expiry and configuration precedence. Distinguish an expired token from an authenticated caller that lacks permission.

Then record action, resource ARN, principal/session ARN, request conditions, and the complete sanitized error. Check identity policies, resource policies, role trust for assume-role failures, session policies, permissions boundaries, applicable SCPs/RCPs, and service-specific access layers. For encrypted data, key permissions may be a separate failure. Endpoint policies can also restrict service access.

**Discriminator:** does the same caller succeed on the same action against an allowed test resource? Does the failure occur before role assumption or after? Use policy tools and audit evidence as aids, while recognizing that simulations may not model every context.

**Repair:** add or correct the narrow required permission/condition at the controlling layer. Avoid attaching administrator access as the default diagnostic step; it can hide the cause and widen risk.

## 2. DNS or network timeout

Start from the actual failing client location. Resolve the hostname and record returned addresses, resolver, and public/private context. Verify the intended port and application listener. Check subnet/route associations, longest-prefix route choice, internet/NAT/endpoint path, security groups, NACLs including return traffic, operating-system firewall, and routing through any inspection appliance.

For hybrid paths, inspect on-premises routes, BGP advertisements where applicable, tunnel status, overlapping ranges, DNS forwarding, and asymmetric routing. Reachability Analyzer and flow logs can help identify network configuration issues, but neither proves full application health.

**Discriminator:** DNS failure, connection refusal, connection timeout, TLS error, and HTTP 403 are different layers. A timeout is not proof of an IAM problem, and a 403 is not proof of a routing problem.

**Repair:** correct the exact route/rule/endpoint/hostname. Preserve intended private access; use an approved private connectivity path instead of making a database public.

## 3. TLS or certificate failure

Check requested hostname against certificate names, validity dates, chain/trust, client clock, TLS version/cipher support, and where TLS terminates. Inspect load-balancer/CDN listener configuration and origin TLS separately. A valid client-to-CDN certificate does not prove CDN-to-origin TLS is correct.

**Discriminator:** compare the hostname used by the client with the configured certificate and SNI behavior. Verify custom-domain certificate placement for the exact service.

**Repair:** correct the certificate, domain association, trust chain, or listener/origin configuration. Do not teach disabling certificate verification as the normal fix.

## 4. ALB unhealthy targets or 5xx errors

Inspect target-group health reasons, registered targets/ports, health path/protocol/matcher, target security group, and application readiness. Verify that the application listens on the intended interface, not only localhost. Check whether a health endpoint itself depends on an unavailable database.

Compare load-balancer-generated errors with target-generated errors. Inspect application logs, connection resets, response time, connection limits, keep-alive/idle timeouts, deployments, and target capacity. A 503 caused by absent healthy targets needs a different fix from an application returning 500.

**Repair:** restore healthy capacity or roll back the bad deployment, then resolve the root cause. Avoid making the health check so permissive that it routes users to broken instances.

## 5. EC2 will not launch, boot, or remain healthy

Separate API launch failure from guest boot failure. For launch failure, inspect permissions, quotas, instance/AZ capacity, subnet IP capacity, AMI architecture, instance type compatibility, key/storage settings, and encrypted-volume key access.

For boot/application failure, inspect status checks, console/system logs, user-data/cloud-init results, disk capacity, package dependencies, instance profile, and required outbound/service connectivity. A launch template can create a running instance whose bootstrap never completed.

**Repair:** correct the template and replace disposable instances through the managed workflow. Preserve evidence before replacement when diagnosing a recurring issue. Persisting manual fixes on one instance leaves the next replacement broken.

## 6. S3 access, upload, or stale-content problems

Check exact bucket/key/case, region/endpoint, caller, policy, ownership mode, encryption configuration, and any access-point/endpoint rules. For presigned requests, verify expiration, signed headers, method, credential lifetime, and clock. Do not publish a presigned URL in a public issue.

For stale reads, distinguish S3's documented object consistency from browser/CDN/application caches, asynchronous replication, and an application writing a different key. For a versioned bucket, inspect versions and delete markers. A delete marker can make a current read look missing while previous versions remain.

**Repair:** correct the relevant access or key/cache behavior. Do not disable Block Public Access to fix a private application's missing role permission. Inspect multipart uploads and historical versions when cleaning up storage costs.

## 7. RDS connection, performance, or failover issue

Check endpoint, port, DNS, subnet/security groups, database status, credentials, TLS requirements, and database-user privileges. Network access, IAM access to the service, and database authentication are distinct.

For slow queries, inspect active sessions, query plans, locks, connection count, CPU/memory, storage latency/throughput, free storage, and replication lag where relevant. Analyze the application connection pool and transaction behavior before scaling the instance. During failover, stale connections and DNS caching can extend application recovery beyond database failover completion.

**Repair:** address the measured bottleneck, index/query/transaction issue, pool settings, or reconnection logic. Test schema and parameter changes; some apply immediately, others require restart or replacement. Confirm data integrity after recovery.

## 8. DynamoDB throttling or inconsistent expectations

Inspect the actual operation, item size, capacity mode, table/index metrics, hot partitions, key distribution, retries, and quotas. A sparse global secondary index can have its own constraints; a scan is not an efficient replacement for missing key design.

For unexpected reads, identify whether the request is a base-table read, index read, cached result, or global-table read, and check supported consistency settings for that exact feature. For conditional-write failures, examine concurrent writers and the intended invariant rather than retrying without understanding the condition.

**Repair:** improve access patterns/partition distribution, use the suitable capacity strategy, adjust indexes or batching, and apply bounded retries for retriable failures. Do not weaken a condition that protects correctness merely to eliminate an error.

## 9. Lambda errors, timeouts, or throttles

Inspect invocation mode: synchronous, asynchronous, or event-source mapping. Each has different retries and failure destinations. Review logs, duration, memory, initialization, concurrency, payload validation, execution role, dependency latency, and network configuration.

A VPC-attached function needs appropriate networking for external/service access; attachment alone does not provide internet egress. A slow database or oversized fan-out can cause rising concurrency, throttling, and repeated downstream work.

**Repair:** fix dependency/network/code problems, tune measured memory/runtime behavior, bound concurrency, and design retries/idempotency for the invocation mode. Raising the timeout can make a backlog worse if the real issue is a saturated dependency.

## 10. Queue backlog, duplicate messages, or DLQ growth

Inspect oldest-message age, arrival/processing rate, visible/in-flight counts, consumer health, permissions, visibility timeout, batching, and failure logs. Approximate queue metrics are useful trends, not an exact real-time accounting ledger.

If the worker receives but never deletes, messages return. If processing exceeds visibility timeout, overlapping work can occur. A poison message may repeatedly fail while healthy messages succeed. A too-low redrive count may send transient failures to the DLQ prematurely.

**Repair:** correct the processing cause, choose a suitable timeout/extension strategy, maintain idempotency, and redrive deliberately after testing. Do not purge a queue to hide an unexplained backlog.

## 11. Container pending, crash-looping, or unable to pull

For ECS, inspect service events and stopped-task reasons; distinguish task execution role from application task role. For EKS, inspect Pod status, events, scheduling constraints, node capacity, image-pull access, CNI address availability, probes, and workload identity.

Check image architecture, digest/tag existence, registry/network access, resource limits, application exit code, secret/config references, filesystem permissions, and listening port. A memory limit can kill a healthy-looking process under load. An aggressive liveness probe can create a restart loop during slow startup.

**Repair:** address the actual scheduling/runtime/network cause and redeploy through versioned configuration. Do not grant cluster administrator to the application to fix a missing narrow permission.

## 12. IaC or pipeline failure

Separate syntax/validation, authentication, planning, resource creation, stabilization, and application validation. Inspect the first meaningful error rather than only the final rollback message. Check resource-name collisions, replacement requirements, quotas, dependencies, permissions, and service availability.

For Terraform/OpenTofu, verify backend access, locking, workspace/state identity, provider versions, and actual drift. For CloudFormation, inspect stack events and retained resources. Do not manually delete state or force-unlock an active operation just to clear a visible error.

**Repair:** correct the definition or environment, review the new plan/change set, and reconcile known manual changes. For delivery failures, preserve the last working artifact and use the documented rollback rather than rebuilding an unknown version.

## 13. Data pipeline incorrect or expensive

Check source completeness, schema/version, timestamps/time zones, duplicate keys, late data, partition values, permissions, and transformation outputs. Compare counts plus business aggregates; row counts alone can miss duplicated or corrupted values.

For cost/performance, inspect scanned bytes, file sizes/formats, partition pruning, skewed joins, shuffle/spill, concurrency, and idle provisioned resources. For streaming, check checkpoint state, retention, event-time logic, consumer lag, and replay assumptions.

**Repair:** quarantine invalid inputs, fix contracts or transformations, and backfill deterministically. Never overwrite the only raw copy before validating the repair if recovery depends on it.

## 14. AI output quality, access, or cost failure

Separate model access/quota errors, retrieval misses, incorrect source data, authorization leakage, prompt/configuration regressions, and tool-execution failures. Inspect a representative evaluation set and the actual retrieved context with appropriate privacy controls.

Track input/output size, repeated tool calls, retries, embedding/retrieval work, model latency, and provisioned idle capacity. A model answering fluently does not prove that its sources or tool arguments are valid.

**Repair:** enforce authorization outside the model, constrain tools, correct retrieval/data problems, improve evaluated prompts or model selection, and bound work. Re-run adversarial and negative tests before releasing a fix.

## 15. Unexpected cost

Identify account, region, service, usage type, time window, tags, and whether the change is price, quantity, retention, or a commitment allocation. Correlate with deployments, load tests, bots, event loops, large scans, replication, and telemetry volume.

Contain a confirmed cause with an appropriate reversible action. Stopping an instance may leave disks and other services billed. Remove only verified lab resources; preserve data needed for recovery or investigation. Recheck later because billing data can arrive after the resource is gone.

## Evidence package for escalation

Provide a concise problem statement, affected resource IDs privately, region/account context, timestamps and time zone, request IDs, sanitized errors, expected versus actual behavior, reproduction steps, recent changes, checks already performed, and impact. Never include secret access keys, session tokens, passwords, or unredacted sensitive data.

Use the official troubleshooting sections linked from the technical chapters and [AWS documentation](https://docs.aws.amazon.com/) to investigate the exact service error rather than treating this broad playbook as a complete error-code reference.
