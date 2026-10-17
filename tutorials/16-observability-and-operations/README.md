# Observability and operations tutorials

[All tutorials](../README.md) · [Source roadmap](../../16-observability-and-operations.md)

Complete these in order. Tutorial 1 uses optional, narrowly scoped AWS resources; tutorial 2 runs entirely locally before its documented AWS extension; tutorial 3 begins with an incident simulation and makes live administration optional. Read the cost and cleanup sections before any deployment. The commands are instructions for your sandbox; creating this documentation did not execute them.

| Tutorial | Result |
| --- | --- |
| [01 — Logs, metrics, alarms, and customer checks](01-logs-metrics-and-alarms.md) | A reproducible incident dataset, queries, an alarm test, and a customer-facing SLI |
| [02 — OpenTelemetry and metrics collection](02-opentelemetry-and-metrics.md) | Correlated spans, a controlled failure, and a collector/backend design |
| [03 — Operate an incident and roll back configuration](03-incident-and-configuration-operations.md) | Tested runbook, change evidence, Systems Manager access plan, and AppConfig rollback design |

## Coverage mapping

| Original substantive heading or branch | Subtopics | Worked location |
| --- | --- | --- |
| Customer journey and signals | Business success, metrics, logs, traces, changes, profiles | 01 dataset and SLI; 02 spans and profiling decision |
| AWS evidence selection | CloudWatch, CloudTrail, Config, Flow Logs, AWS Health | 01 queries; 03 evidence timeline |
| Metrics and logs | Dimensions, percentiles, cardinality, redaction, retention, Logs Insights | 01 complete exercise |
| OpenTelemetry | SDK, collector, context, sampling, buffering, X-Ray lifecycle | 02 complete exercise |
| Prometheus/Grafana | AMP, AMG, PromQL, labels, dashboard permissions | 02 export and backend extension |
| Service objectives and alerting | SLI/SLO/SLA, error budget, burn rate, missing data, notifications | 01 calculations and alarm |
| Customer monitoring from coverage map | CloudWatch Synthetics and RUM | 01 browser/canary extension |
| Safe operations | Session Manager, Run Command, State Manager, Patch Manager, Automation, inventory, parameters | 03 controlled change workflow |
| Configuration rollout from coverage map | AppConfig validation, progressive rollout, alarms, Parameter Store distinction | 03 configuration exercise |
| Incidents and improvement | Roles, playbooks, runbooks, timelines, detection/restoration time, toil | 03 incident simulation |
| Practical lab, pitfalls, mastery | Fault, diagnosis, recovery, evidence, cost, cleanup | All three acceptance checklists |

Use synthetic records and dedicated resource names. Keep request IDs in logs and traces rather than metric dimensions. An alarm notification test should go only to a destination you control. Use current service-specific instructions before a live extension; X-Ray SDK/daemon maintenance and Incident Manager onboarding restrictions are explained in the source chapter.
