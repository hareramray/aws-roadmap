# Build a useful dashboard from a broken checkout

[Domain index](README.md) · [All tutorials](../README.md)

This exercise starts with a complete synthetic incident dataset. You will calculate customer impact, investigate structured logs, and optionally publish a tiny sample to CloudWatch. No deployed application is required. Prerequisites are Python, the AWS CLI for the optional extension, and the permissions/identity checks in the setup module.

## 1. Define the operation and budget

The operation is an eligible checkout attempt. Success means a durable order was accepted, not merely that a web server returned HTTP 200. Use a fictional request-based objective of 99.5% successful eligible attempts over 30 days. Treat malformed input as ineligible, and count dependency timeouts as failures. These are exercise definitions rather than service guarantees.

Create a local working directory. Save the following as `make_events.py`; it generates no personal data and makes no network requests.

```python
import json
from pathlib import Path

events = []
for i in range(1000):
    eligible = i < 980
    failed = eligible and 400 <= i < 420
    event = {
        "timestamp": 1792272000000 + i * 100,
        "service": "checkout",
        "environment": "tutorial",
        "route": "/orders",
        "request_id": f"request-{i:04d}",
        "deployment": "v2" if 400 <= i < 700 else "v1",
        "eligible": eligible,
        "outcome": "invalid" if not eligible else "failed" if failed else "accepted",
        "duration_ms": 1200 if failed else 80 + i % 40,
        "error_code": "PAYMENT_TIMEOUT" if failed else None,
    }
    events.append(event)
Path("checkout.jsonl").write_text(
    "\n".join(json.dumps(e) for e in events), encoding="utf-8")
eligible = [e for e in events if e["eligible"]]
good = sum(e["outcome"] == "accepted" for e in eligible)
print({"eligible": len(eligible), "good": good,
       "failed": len(eligible) - good,
       "availability": good / len(eligible)})
```

Run `python make_events.py`. Expect 980 eligible attempts, 960 successful attempts, 20 failures, and availability approximately `0.979592`. The short experiment does not establish a 30-day SLO result; it demonstrates the calculation on the available sample.

The allowed bad fraction is `1 - 0.995 = 0.005`. The observed bad fraction is `20 / 980 ≈ 0.020408`; the observed burn rate is about `4.08`. Sustaining that rate would consume error budget roughly four times faster than planned. For an operational alert, combine a fast window with a slower confirmation window and a meaningful traffic threshold.

## 2. Investigate locally before opening a dashboard

Add a second script or use a Python REPL to group events by deployment and error code. You should find all twenty failures in deployment `v2`, with `PAYMENT_TIMEOUT`. That correlation identifies a hypothesis, not proof that the deployment caused the issue; an external payment outage could have occurred simultaneously.

Sort eligible durations and inspect the 95th and 99th percentile using a documented percentile method. The tail differs from the average because only a small proportion of requests time out. Record the calculation method and sample size; do not compare dashboard percentiles produced by different aggregations as if they were identical.

Write the first diagnostic question precisely: “Did the new deployment change payment timeout behavior?” Necessary next evidence includes a configuration diff, dependency latency, and traces. Merely adding more CPU graphs would not resolve that uncertainty.

## 3. Optional CloudWatch Logs upload

Choose one sandbox Region and profile, verify the account, and estimate log ingestion, storage, queries, custom metrics, and alarms. Set a one-day lab retention where supported. Do not upload the historical timestamps unchanged: some log APIs reject events outside their accepted age window.

```powershell
$Profile = "roadmap"
$Region = "ap-south-1"
$LogGroup = "/tutorial/checkout-observability"
aws sts get-caller-identity --profile $Profile --region $Region
aws logs create-log-group --log-group-name $LogGroup --profile $Profile --region $Region
aws logs put-retention-policy --log-group-name $LogGroup --retention-in-days 1 --profile $Profile --region $Region
aws logs create-log-stream --log-group-name $LogGroup --log-stream-name sample --profile $Profile --region $Region
```

Prepare a small upload using Python; CloudWatch's event timestamp is current while the original event time remains a data field for analysis.

```python
import json, time
from pathlib import Path
records = [json.loads(s) for s in Path("checkout.jsonl").read_text().splitlines()]
now = int(time.time() * 1000)
payload = [{"timestamp": now + i,
            "message": json.dumps(record)} for i, record in enumerate(records)]
Path("log-events.json").write_text(json.dumps(payload), encoding="utf-8")
```

Upload with `aws logs put-log-events --log-group-name $LogGroup --log-stream-name sample --log-events file://log-events.json --profile $Profile --region $Region`. This payload is intentionally small; production clients must obey batch size, timestamp ordering, and request limits.

In Logs Insights select only this group and the upload interval. Run:

```text
fields deployment, outcome, error_code, duration_ms
| filter eligible = true
| stats count(*) as attempts, pct(duration_ms, 99) as p99 by deployment, outcome
```

Then inspect the failures:

```text
fields @timestamp, request_id, deployment, error_code
| filter outcome = "failed"
| sort @timestamp asc
| limit 25
```

Expect twenty failure rows. If none appear, check group, Region, query window, JSON field discovery, and the API upload result before changing application code.

## 4. Publish a bounded metric and test an alarm

Use the CloudWatch console or CLI to publish `EligibleAttempts=980` and `FailedAttempts=20` under namespace `Tutorial/Checkout`, with dimensions `Service=checkout` and `Environment=tutorial`. Use Count units and a common timestamp. Do not add `request_id` as a dimension.

Create metric math `100 * failures / attempts`, with both inputs using Sum over the same period. Define no-traffic handling deliberately: do not interpret division by zero as healthy traffic. Create a temporary alarm on an exercise threshold above the normal baseline, configure evaluation periods, and route it only to a notification destination you control. Confirm the alarm transitions because of real metric data, then publish a later healthy interval and verify recovery. Changing an alarm's state manually tests part of notification plumbing but does not test the metric expression.

Add dashboard panels for attempts, failure percentage, p99 duration, deployment markers, and oldest pending order if the real application is asynchronous. Each panel must answer a question an operator actually uses.

## 5. Add customer checks, not just server checks

For a later real application, a CloudWatch Synthetics canary can periodically execute a synthetic checkout journey. Give it a dedicated synthetic tenant, bounded permissions, and cleanup of created records. Verify both successful completion and a deliberate authorization failure. A canary can incur charges even without real users.

CloudWatch RUM observes supported real browser experiences: page loading, errors, and user-facing performance. Decide collection, sampling, consent, sensitive URL treatment, and retention before enabling it. A healthy canary and failing real users can both be true because geography, browsers, and authentication paths differ. Compare Synthetics and RUM rather than treating either as exhaustive coverage.

## Verification, troubleshooting, and cleanup

Keep the generated input, exact SLI definition, query results, alarm transition, and one diagnosis hypothesis with evidence. Reject a dashboard that shows 100% success because invalid requests or timeouts disappeared from the denominator.

If alarms stay insufficient-data, inspect metric dimensions, timestamps, periods, and missing-data treatment. If queries cost too much, shorten the time range and select fewer log groups before removing useful evidence. If alerts repeat with no action, revise the threshold or route them to a nonurgent work queue.

Delete the temporary alarm, dashboard, custom collection jobs, canary, RUM app monitor if created, notification subscriptions, and dedicated log group. Custom metric data ages out according to the service rather than being deleted like a log group. Stop publishing it. Remove local files only after retaining desired evidence.

References: [CloudWatch metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html), [Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html), [Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html), [RUM](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html).
