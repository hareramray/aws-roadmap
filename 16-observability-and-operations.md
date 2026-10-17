# 16. Observability and operations

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete [account setup](02-account-and-tooling.md), [IAM](04-iam-and-identity.md), one compute module, and [infrastructure as code](12-infrastructure-as-code.md). You should understand HTTP status codes, JSON, timestamps, application exceptions, and basic command-line troubleshooting.

By the end, you should be able to instrument a workload, explain a failed customer request, define useful service objectives, build actionable alarms, operate safely through Systems Manager, and produce an incident report. Monitoring asks whether a known condition occurred; observability provides enough evidence to investigate conditions you did not predict. Neither is complete without an owner who can respond.

## 1. Start with the customer journey

Select an operation such as placing an order. Write its success condition, latency expectation, dependency chain, and business consequence of failure. An HTTP 200 response is insufficient if the order was never persisted. A healthy EC2 CPU graph says little about whether payment processing works.

Define the signals before selecting dashboards:

- **Metrics:** numerical time series for request rate, errors, duration, saturation, queue age, replication lag, and domain outcomes.
- **Logs:** discrete events with enough context to reconstruct a decision or failure.
- **Traces:** spans showing how one operation crossed processes and dependencies.
- **Events and changes:** deployments, configuration changes, scaling, maintenance, and security actions that explain changes in behavior.
- **Profiles:** sampled resource consumption that helps explain where CPU or memory is spent.

Distinguish symptoms from causes. An elevated checkout failure rate is a customer symptom; database connection saturation is a possible cause. Page on urgent symptoms and use cause metrics to diagnose. Batch pipelines also need freshness, completeness, and correctness signals even when no HTTP endpoint exists.

## 2. Know which AWS evidence answers which question

| Tool or evidence | Main question | Important limitation |
| --- | --- | --- |
| Amazon CloudWatch | How is the application or infrastructure behaving? | Collection, dimensions, retention, and alarm semantics require design |
| AWS CloudTrail | Which recorded AWS API activity occurred, by whom, and from where? | Management, data, and other event categories have different configuration and costs |
| AWS Config | What supported resource configuration existed, and did it satisfy a rule? | Configuration evaluation is not an application trace or a universal prevention mechanism |
| VPC Flow Logs | Which network flows were accepted or rejected? | They do not capture application payloads |
| AWS Health | Which AWS events may affect my resources? | It complements application checks rather than proving customer health |
| Application traces | Which dependency or code span consumed time or failed? | Sampling and missing instrumentation can hide individual requests |

Practice correlating the same incident across these sources. A security-group change may appear in CloudTrail, a configuration change in Config, rejected traffic in flow logs, and failures in application telemetry. Time synchronization, account IDs, Regions, resource IDs, and deployment versions make the correlation reliable.

## 3. Build metrics and logs deliberately

Learn namespaces, dimensions, units, aggregation periods, statistic selection, metric math, and percentiles. Average latency can hide a poor tail; p95 and p99 reveal different experiences, but small sample counts need careful interpretation. A CPU percentage averaged across many instances can hide one saturated node.

Keep metric dimensions bounded. Request IDs, customer emails, and arbitrary URLs belong in carefully controlled logs or traces, not unbounded metric dimensions. Normalize a route to `/orders/{id}` instead of emitting a separate series for every order. Cardinality increases storage, ingestion, query work, and operational complexity.

Use structured JSON logs with a schema such as:

```json
{"level":"ERROR","service":"orders","environment":"lab","operation":"create_order","request_id":"sample-123","trace_id":"example","deployment":"v1","error_code":"DEPENDENCY_TIMEOUT","duration_ms":840}
```

Never include passwords, tokens, payment data, or full request bodies by default. Establish redaction, access policies, encryption, retention, and deletion requirements before sending production records. Different log classes need different retention. Keep enough evidence for investigation without retaining verbose debug output indefinitely.

Learn CloudWatch Logs Insights filtering, aggregation, parsing, sorting, and time-window selection. Begin investigations with a small time interval and a specific service. Broaden only when necessary; scanning every log group for every question increases cost and noise. Track dropped records and collector health so an empty dashboard cannot be mistaken for a healthy system.

## 4. Instrument with OpenTelemetry

Learn OpenTelemetry resources, semantic conventions, spans, attributes, context propagation, exporters, and collectors. A trace should preserve context across HTTP, queue messages, worker processes, and asynchronous callbacks where supported. Auto-instrumentation accelerates coverage, while manual spans capture business steps that libraries cannot understand.

Use AWS Distro for OpenTelemetry or supported CloudWatch instrumentation as appropriate for the runtime. Understand head sampling versus tail sampling, sampling bias, collector buffering, backpressure, retry limits, and the consequences of losing an exporter. Telemetry must not take down the business service when a collector is unavailable.

For new learning, prefer OpenTelemetry over building around legacy X-Ray SDKs. AWS states that the X-Ray SDKs and daemon entered maintenance mode on February 25, 2026 and recommends OpenTelemetry migration; this is a lifecycle change to instrumentation, not a statement that the X-Ray tracing service disappeared. [AWS X-Ray daemon guidance](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon.html).

Compare CloudWatch dashboards and Application Signals with Amazon Managed Service for Prometheus and Amazon Managed Grafana. CloudWatch is a natural starting point for AWS service metrics; Prometheus is useful for established PromQL and Kubernetes metric practices; Grafana provides cross-source visualization. Avoid duplicating every signal across every backend without a concrete need.

## 5. Define service objectives and alerting

A **service level indicator (SLI)** is a measured behavior; an **SLO** is the target over a defined interval; an **SLA** is a contractual commitment. Example: 99.5% of valid order attempts complete successfully within a rolling 30-day window. That target is an exercise assumption, not an AWS guarantee.

For a request-based availability SLO:

```text
availability = good eligible requests / all eligible requests
error budget = 1 - SLO target
burn rate = observed bad-request fraction / allowed bad-request fraction
```

Define what counts as eligible. Decide how timeouts, client errors, retries, partial completion, planned maintenance, and no-traffic intervals are treated. Request-based and period-based objectives answer different questions. Application Signals supports both approaches, and its default availability semantics may differ from your business definition. [CloudWatch SLO documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-ServiceLevelObjectives.html).

Learn alarm evaluation periods, datapoints to alarm, missing-data behavior, composite alarms, anomaly detection, and notification delivery. Use sustained conditions and appropriate windows to reduce flapping. Every page needs an owner, severity, dashboard, runbook, and clear response. A warning that can wait until business hours should become a ticket or report instead of an emergency page.

## 6. Operate safely and learn from incidents

Study Systems Manager Session Manager, Run Command, State Manager, Patch Manager, Automation, inventory, and parameter handling. Understand the agent, instance role, network endpoints, and logging prerequisites. Prefer audited role-based access to unmanaged SSH keys, but verify the exact session logging support for the session type you use.

Study **AWS AppConfig** for controlled application-configuration and feature-flag changes. Define applications, environments, configuration profiles, validators, and deployment strategies. Validate syntax and business constraints before a rollout; expose changes gradually and configure CloudWatch alarms that can trigger rollback during deployment monitoring. Parameter Store supplies configuration values, while AppConfig adds a managed validation/deployment workflow and can use Parameter Store as a source. Plan retrieval caching, IAM access, and metered requests. [AWS AppConfig overview](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html).

As an optional extension to an existing sandbox app, deploy a harmless feature flag through a staged strategy. Verify that invalid configuration is rejected, observe which configuration version the app uses, and rehearse an alarm-triggered rollback with a synthetic signal. Record propagation and recovery time; remove the test deployment/configuration and any dedicated alarm afterward. A flag controlling presentation is not a substitute for server-side authorization.

A runbook describes a repeatable task; a playbook guides investigation when the cause is uncertain. Both need prerequisites, permissions, expected observations, stop conditions, rollback, and ownership. Automation should have concurrency limits, bounded scope, idempotency, and verification. Restarting every instance may temporarily hide a memory leak while worsening an outage.

Prepare incident roles: incident lead, technical responders, communications owner, and note taker. Establish an incident timeline, record hypotheses and evidence, restore service, then investigate causes. Write blameless follow-ups with specific owners and due dates. Track detection time, restoration time, recurrence, alert quality, and operational toil without turning metrics into individual blame.

Do not make a new lab depend on Systems Manager Incident Manager: AWS stopped accepting new customers on November 7, 2025. Learn the incident-management process and integrate an available notification/on-call system when needed. [Availability change](https://docs.aws.amazon.com/incident-manager/latest/userguide/incident-manager-availability-change.html).

## Practical lab: investigate an intentionally broken request

**Scope:** one disposable API or worker from an earlier module, a tiny synthetic workload, and one Region. Use only synthetic data.

1. Record a cost estimate for logs, custom metrics, traces, alarms, and optional synthetic checks. Set explicit retention and an experiment end time.
2. Add structured logs and a correlation ID. Instrument one dependency with OpenTelemetry where practical.
3. Produce a baseline with successful and failed requests. Build a dashboard showing rate, errors, duration, and one saturation metric.
4. Create a failure-rate alarm with a minimum traffic condition or another appropriate low-traffic treatment. Route it to a notification endpoint you control.
5. Introduce a reversible application-level timeout or controlled error in the lab. Do not disrupt shared infrastructure.
6. Start a timer when the fault begins. Use the alarm, logs, and trace to identify the failing dependency and deployment version. Record detection and diagnosis times.
7. Restore the configuration and confirm recovery. Write a one-page incident record and improve one missing signal.

**Verification:** demonstrate an actual notification, one request trace or correlation trail, the precise error count, and recovery to the baseline. Explain what would happen if telemetry stopped arriving. Test the runbook from a clean terminal rather than relying on remembered console steps.

**Cleanup:** remove the fault toggle, test traffic, alarms, dashboards, schedules, collectors, temporary IAM grants, and lab resources. Delete or shorten retention for disposable log groups. Retained traces, logs, managed workspaces, and synthetic checks can continue to incur charges.

## Common pitfalls

- Collecting everything before deciding what decisions the evidence should support.
- Logging secrets and assuming encryption makes their collection acceptable.
- Using CPU utilization as the only scaling or customer-health signal.
- Treating an alarm's missing data as universally good or universally bad.
- Paging repeatedly without ownership, response instructions, or a customer consequence.
- Ignoring telemetry ingestion failures, notification failures, and monitoring-account access.
- Assuming tracing samples prove that no other failures occurred.

## Mastery checklist

- [ ] I can distinguish CloudWatch, CloudTrail, Config, flow logs, and traces.
- [ ] I can explain p99 latency, cardinality, sampling, and retention tradeoffs.
- [ ] I can follow a request across asynchronous boundaries.
- [ ] I can define an SLI whose semantics match a business operation.
- [ ] I can demonstrate a tested alarm and a usable recovery runbook.
- [ ] I can write an incident analysis with evidence and prevention actions.
- [ ] I can estimate and reduce observability costs without removing essential evidence.

## Official references and next step

- [CloudWatch documentation](https://docs.aws.amazon.com/cloudwatch/)
- [CloudTrail documentation](https://docs.aws.amazon.com/cloudtrail/)
- [AWS Config documentation](https://docs.aws.amazon.com/config/)
- [AWS Distro for OpenTelemetry](https://aws-otel.github.io/)
- [Systems Manager documentation](https://docs.aws.amazon.com/systems-manager/)

Continue to [reliability and disaster recovery](17-reliability-and-disaster-recovery.md), where these signals become evidence that recovery actually works.
