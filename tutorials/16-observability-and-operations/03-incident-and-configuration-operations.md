# Diagnose a configuration incident and rehearse safe recovery

[Domain index](README.md) · [All tutorials](../README.md)

This is a worked incident simulation with optional AWS implementation. It teaches how to distinguish service health, API audit records, resource configuration, network evidence, and application behavior. You will also design a controlled AppConfig rollout and a Systems Manager runbook. Do not change a shared environment to manufacture an incident.

## 1. Assemble the incident packet

Create `incident.csv` with these synthetic observations. The times are UTC and deliberately close enough to require ordering by timestamp rather than recollection.

```csv
time,source,observation
10:00:00,change,configuration version 2 begins rollout
10:01:00,application,payment timeout changed from 1500ms to 100ms
10:02:00,metric,checkout failure fraction rises from 0.1% to 8%
10:02:20,alarm,availability burn-rate alarm fires
10:03:00,trace,payment dependency p95 remains 300ms
10:03:20,network,no corresponding rise in rejected flows
10:04:00,operator,configuration rollback requested
10:05:00,application,configuration version 1 active
10:06:00,metric,failure fraction returns below 0.2%
```

State the business symptom: eligible checkouts fail because the application times out before an otherwise normal dependency response. The unchanged dependency p95 and the configuration change support this explanation. They do not prove every failure has the same cause; inspect representative failures and competing explanations.

Calculate detection time as 20 seconds from the chosen impact start at 10:02:00 to the alarm at 10:02:20. Calculate recovery time as four minutes from impact start to the observed return at 10:06:00. Record the definitions explicitly; measuring from when an operator opened the ticket would hide detection delay.

## 2. Choose evidence for each hypothesis

Build this investigation table before opening tools:

| Hypothesis | Best evidence | What a negative result does not prove |
| --- | --- | --- |
| A configuration change shortened the timeout | Deployment/configuration history, application version logs | A clean infrastructure diff does not rule out application configuration |
| An AWS API action changed access | Relevant CloudTrail events | Event history does not include every data event by default |
| A security group or route changed | Config history and CloudTrail | A supported-resource record is not full packet visibility |
| Traffic was blocked | Flow Logs and endpoint/host evidence | Accepted flow metadata does not prove application success |
| An AWS service event affected dependencies | AWS Health plus local application measurements | No public event does not guarantee the workload is healthy |
| The payment service became slow | Traces and dependency metrics | A trace sample may miss unsampled failures |

For a live sandbox, note account, Region, resource ID, request ID, and time window for each source. Use current CloudTrail trails and supported query paths; do not require a new CloudTrail Lake account. Record who changed what and why, without treating an operator identity as proof of malicious intent.

## 3. Validate a configuration before rollout

Save this candidate configuration as `checkout-config.json`:

```json
{"payment_timeout_ms":100,"max_attempts":2,"feature_enabled":true}
```

Use a local validator to express an operational requirement. Here the lower timeout limit is an exercise assumption supported by the observed dependency behavior, not a universal AWS value.

```python
import json
from pathlib import Path

c = json.loads(Path("checkout-config.json").read_text())
errors = []
if not isinstance(c.get("payment_timeout_ms"), int):
    errors.append("payment_timeout_ms must be an integer")
elif not 500 <= c["payment_timeout_ms"] <= 5000:
    errors.append("payment_timeout_ms must be within 500..5000")
if c.get("max_attempts") not in (1, 2, 3):
    errors.append("max_attempts must be 1, 2, or 3")
if type(c.get("feature_enabled")) is not bool:
    errors.append("feature_enabled must be boolean")
if errors:
    raise SystemExit("INVALID: " + "; ".join(errors))
print("VALID")
```

Expect the candidate to fail. Change the timeout to 1500 and expect `VALID`. Also test a string timeout, missing field, and excessive retries. A type/schema validator can enforce shape; semantic validation checks relationships and operational constraints. Neither guarantees the configuration performs well under real load.

## 4. Map the process to AWS AppConfig

AppConfig manages application configuration deployment using applications, environments, configuration profiles, validators, deployment strategies, and optional alarm-based rollback. Parameter Store stores configuration and supported encrypted parameters; fetching a new value from it does not by itself provide a staged rollout or health-based rollback.

For an optional implementation, create a dedicated AppConfig application and tutorial environment. Choose a supported hosted configuration or external source, attach a compatible JSON Schema or Lambda validator, and create a valid version. Use a gradual deployment strategy with an observation interval and a CloudWatch alarm that represents customer failure. Confirm the required monitoring role and permissions using current documentation.

Integrate the application through the supported AppConfig agent or data retrieval path rather than polling management APIs on every customer request. Cache configuration safely, handle retrieval outages, record active version, and decide startup behavior when no valid configuration exists.

Rehearse a valid change, then a schema-invalid version. The latter should be rejected before rollout. A separately constructed, schema-valid but behaviorally bad version can test rollback in an isolated application; bound its exposure and use synthetic traffic. Verify both rollback initiation and the application's observed return to the previous configuration. An alarm entering ALARM does not prove every client has refreshed its cache yet.

## 5. Turn administration into a runbook

Prepare a runbook for inspecting and restarting one disposable service. Include scope, prerequisites, exact target tag, required role, expected output, timeout, stop conditions, rollback, and post-action health verification.

For Systems Manager, distinguish the tools:

- Session Manager gives supported interactive access; verify agent, IAM, network, and session logging support.
- Run Command executes a bounded command on explicitly selected managed nodes.
- Automation coordinates a multi-step operational procedure and can validate before and after actions.
- State Manager enforces intended configuration repeatedly; ensure repeated execution is safe.
- Patch Manager supports patch operations and compliance workflows; establish maintenance, reboot, and validation behavior.
- Inventory provides supported software/system information; it is not a runtime vulnerability guarantee.

Do not target `*` or a broad production tag to practice Run Command. Begin with one known tutorial instance and inspect its identity before execution. A command that reports success because a process restarted is incomplete unless the customer-facing health check recovers.

Write an approval point only where the organization's established process requires one; the technical runbook should still be concrete and reviewable. Do not create a new dependency on Incident Manager for this exercise because new-customer onboarding is restricted.

## 6. Conduct the incident drill

Assign incident lead, technical investigator, communications owner, and note taker; one learner can simulate the roles with separate notes. Begin with the alarm packet, keep a timestamped hypothesis log, choose the smallest reversible mitigation, and verify the recovery signal.

Produce a concise incident report: customer effect, timeline, contributing conditions, recovery action, evidence, and follow-ups. Useful follow-ups are “reject timeout below 500ms in validator” and “show active configuration version on dashboard,” each with an owner and acceptance test. “Be more careful” is not a testable corrective action.

## Verification, cost, and cleanup

Your evidence must include the rejected invalid configuration, a successful valid configuration, a rollback plan, a clear mapping of evidence sources, and a measured simulated detection/recovery interval. Label all simulated timings as simulated; do not advertise them as production recovery results.

Optional live costs include AppConfig requests/deployments, Lambda validation, CloudWatch alarms/logs, Config recording, trail storage/events, and managed instances/network endpoints. Avoid enabling fleet-wide recording merely for one tutorial.

Delete dedicated AppConfig resources and stop its polling agent after the application is removed. Remove the temporary alarm, validator function, runbook, IAM grants, log groups, and dedicated instance only if created for this lab. Keep shared organization trails, Config recorders, and operational baselines intact. Retain redacted runbooks and incident evidence locally.

References: [AWS AppConfig](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html), [Systems Manager](https://docs.aws.amazon.com/systems-manager/), [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html), [AWS Config](https://docs.aws.amazon.com/config/), [AWS Health](https://docs.aws.amazon.com/health/).
