# Promote a canary and recover a compatible release

[Delivery tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../13-devops-and-delivery.md)

**Outcome:** Demonstrate a backward-compatible data migration locally, then optionally route a small fraction of Lambda invocations to a candidate version and restore the previous version. Complete the [build tutorial](01-secure-build-and-release-pipeline.md). Cloud sections require an existing dedicated sandbox function and are billable; the migration exercise uses only a temporary local SQLite database.

## 1. Prove that data permits rollback

Application rollback is safe only if the earlier release can still read data written by the newer release. Save `migration.py`:

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE orders (id INTEGER PRIMARY KEY, amount_cents INTEGER NOT NULL)")
db.execute("INSERT INTO orders VALUES (1, 250)")

def read_v1():
    return db.execute("SELECT id, amount_cents FROM orders ORDER BY id").fetchall()

assert read_v1() == [(1, 250)]
# Expand: keep the old representation available.
db.execute("ALTER TABLE orders ADD COLUMN currency TEXT")
db.execute("UPDATE orders SET currency='USD' WHERE currency IS NULL")
# New release writes both the compatible amount and the additional field.
db.execute("INSERT INTO orders (id, amount_cents, currency) VALUES (2, 375, 'USD')")
assert read_v1() == [(1, 250), (2, 375)]
new_rows = db.execute("SELECT id, amount_cents, currency FROM orders ORDER BY id").fetchall()
assert new_rows == [(1, 250, "USD"), (2, 375, "USD")]
print("PASS: old and new readers work after expansion")

# Controlled bad contract change: remove the old reader's expected column.
db.execute("ALTER TABLE orders RENAME COLUMN amount_cents TO amount_minor_units")
try:
    read_v1()
except sqlite3.OperationalError:
    print("EXPECTED FAILURE: old release cannot read the contracted schema")
else:
    raise AssertionError("The incompatible change should break v1")
db.close()
```

Run `python .\migration.py`. Expect one PASS and one expected failure. The in-memory database disappears on exit. In a real system, expand the schema, deploy compatible readers/writers, backfill with reconciliation, observe adoption, then contract only after the rollback window and old consumers are retired. Reversing a deployment does not reverse a destructive migration or repair payments already issued. Backups need a tested restore and an accepted recovery-point objective.

## 2. Define release evidence before traffic moves

Record the stable version, candidate version, immutable artifact digest, configuration, test results, owner, and stop conditions. Choose measurable thresholds for error rate, latency, business correctness, and dependency load. Tiny samples cannot reliably detect rare regressions. A canary is a controlled experiment with an exposure budget, not a fixed percentage that guarantees safety.

Feature flags separate code deployment from behavior exposure. Scope a flag to a known test cohort, verify both states, record its owner/expiry, and provide a reliable off switch. Do not put secrets in a client-visible flag. A flag rollback cannot reverse incompatible data already written.

## 3. Prepare two immutable Lambda versions

Use the previous tutorial's function, with version 1 representing the reviewed correct code and a second published version built from a deliberate candidate change. Numeric versions are allocated by AWS; never assume they are literally `1` and `2`. Record actual values as `$env:LAB_STABLE_VERSION` and `$env:LAB_CANDIDATE_VERSION`, and the function name as `$env:LAB_FUNCTION`.

For a useful controlled failure, make candidate code return an incorrect total for quantity 2 while retaining its validation. The unit test should detect this; bypassing that test is permitted only in this isolated failure drill with synthetic inputs, and must not become a delivery practice. Alternatively, inject a known error using a test-only configuration path. Record that the bad version is intentionally ineligible for promotion.

Verify both versions directly with `aws lambda invoke --function-name ... --qualifier <version> --cli-binary-format raw-in-base64-out --payload file://request.json response.json`, using a UTF-8 request file `{"quantity":2}`. Inspect CLI metadata, `FunctionError`, and response contents. Transport success alone is not application success. For weighted routing, AWS requires compatible version settings, including the same execution role and dead-letter queue configuration.

## 4. Create the alias and route a small canary

Create a dedicated alias `roadmap-live` initially pointing entirely to the stable version. If it already exists, inspect its configuration first instead of overwriting an unknown deployment. Confirm caller account/Region and check every native exit code.

```powershell
aws lambda create-alias --function-name $env:LAB_FUNCTION --name roadmap-live --function-version $env:LAB_STABLE_VERSION
if ($LASTEXITCODE -ne 0) { throw 'Alias creation failed; inspect whether the alias already exists' }
$weights = @{}
$weights[$env:LAB_CANDIDATE_VERSION] = 0.1
$routing = @{ AdditionalVersionWeights = $weights } | ConvertTo-Json -Depth 4
[IO.File]::WriteAllText((Join-Path $PWD 'routing.json'), $routing, [Text.UTF8Encoding]::new($false))
aws lambda update-alias --function-name $env:LAB_FUNCTION --name roadmap-live --function-version $env:LAB_STABLE_VERSION --routing-config file://routing.json
if ($LASTEXITCODE -ne 0) { throw 'Canary update failed' }
```

In a concurrent delivery environment, read the alias revision ID and pass the corresponding update precondition to avoid overwriting another release. The example weight is a learning choice; choose an exposure budget based on risk and traffic. Actual distribution is probabilistic and can differ noticeably for small samples.

## 5. Measure a bounded synthetic batch

Save `probe.py`. This optional script invokes only when explicitly passed `--invoke`, checks the expected account, sends at most 30 requests, and records the executed version and business result. It uses an already installed compatible Boto3 and configured credentials.

```python
import argparse
import collections
import json
import os
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("--invoke", action="store_true")
args = parser.parse_args()
if not args.invoke:
    print("Preview: 30 synchronous synthetic invocations of alias roadmap-live")
    raise SystemExit(0)

import boto3
expected = os.environ["EXPECTED_AWS_ACCOUNT"]
if len(expected) != 12 or not expected.isdigit():
    raise ValueError("Set the expected sandbox account ID")
session = boto3.Session(region_name=os.environ.get("AWS_REGION") or os.environ.get("AWS_DEFAULT_REGION"))
if not session.region_name:
    raise ValueError("Set an explicit sandbox Region or a profile with a Region")
if session.client("sts").get_caller_identity()["Account"] != expected:
    raise RuntimeError("Wrong account")
client = session.client("lambda")
records = []
try:
    for _ in range(30):
        response = client.invoke(FunctionName=os.environ["LAB_FUNCTION"],
                                 Qualifier="roadmap-live", InvocationType="RequestResponse",
                                 Payload=json.dumps({"quantity": 2}).encode())
        body = response["Payload"].read()
        response["Payload"].close()
        value = json.loads(body)
        good = not response.get("FunctionError") and value.get("total_cents") == 250
        records.append({"version": response.get("ExecutedVersion"), "good": good})
finally:
    client.close()
    Path("probe-results.json").write_text(json.dumps(records, indent=2), encoding="utf-8")
print(dict(collections.Counter(row["version"] for row in records)))
print({"requests": len(records), "business_failures": sum(not row["good"] for row in records)})
```

A sample may miss the candidate entirely; do not claim zero observed failures proves it is safe. Direct-version tests establish the candidate's known defect. CloudWatch error metrics alone may miss an incorrect successful response, so inspect the business invariant too. Add latency/error alarms with explicit missing-data behavior and enough observations for a useful decision. Provisioned concurrency, if added, changes cost and capacity planning and requires separate cleanup.

## 6. Restore, verify, and decide promotion

To roll back, point the alias to the recorded stable version and remove additional weights. Save `{"AdditionalVersionWeights":{}}` to `clear-routing.json`, then run:

```powershell
aws lambda update-alias --function-name $env:LAB_FUNCTION --name roadmap-live --function-version $env:LAB_STABLE_VERSION --routing-config file://clear-routing.json
if ($LASTEXITCODE -ne 0) { throw 'Rollback failed; investigate immediately' }
aws lambda get-alias --function-name $env:LAB_FUNCTION --name roadmap-live
if ($LASTEXITCODE -ne 0) { throw 'Alias verification failed' }
```

Verify no candidate weight remains and rerun a bounded probe. All executed versions should now be stable, allowing for any already in-flight invocation. A correct candidate can later become the primary version with no extra weights after approval/evidence. Keep the prior version through the rollback window; do not delete it immediately after promotion.

For ECS, promote the same image digest using the supported rolling/blue-green controller and health/rollback settings; see [ECS service](../11-containers-and-kubernetes/01-ecs-container-service.md). For Kubernetes, inspect Deployment revisions, readiness, surge/unavailable budgets, and database compatibility; see [EKS operations](../11-containers-and-kubernetes/02-kubernetes-and-eks-operations.md). CodeDeploy/SAM can automate supported Lambda traffic shifting and alarms, but validate the generated aliases, hooks, and rollback permissions. Blue/green doubles some capacity temporarily; rolling updates mix versions; immutable replacement improves reproducibility but still needs state compatibility.

## 7. Teardown and incident record

Delete the dedicated alias, candidate/stable versions when unused, sandbox function, alarms, and retained log groups. Remove any provisioned concurrency, scheduled probes, pipeline trigger, and artifact storage created only for this exercise. Keep a sanitized incident note with detection time, exposed requests, wrong behavior, rollback action, verification, and prevention change. CI permission to publish is different from operational permission to stop or restore a release; test the recovery identity before relying on it.

Explain why a 10% configured weight need not produce exactly three candidate executions in 30 calls, why HTTP success misses a price defect, and why contracting a database schema can make application rollback impossible.

Official references: [Lambda weighted aliases](https://docs.aws.amazon.com/lambda/latest/dg/configuring-alias-routing.html), [Lambda immutable versions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html), [CodeDeploy deployment configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html), [safe deployments with SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/automating-safe-deployments.html).
