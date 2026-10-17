# 02 — Read APIs carefully, paginate, and manage asynchronous resources

[Account tutorials](README.md) · [All tutorials](../README.md)

**Mode:** local code plus read-only AWS examples and a known tiny sandbox lifecycle. **Result:** correct handling of profiles, JSON, pagination, errors, polling, idempotency, and cleanup. The account check from tutorial 01 comes first.

## 1. Inspect the request shape

Open the official CLI reference for the operation you intend to use. Identify required inputs, resource scope, optional flags, pagination, output fields, and error conditions. The same English word can refer to different API and IAM action names, so verify authorization separately.

List only tagged lab instances in the intended account/region:

```powershell
aws ec2 describe-instances --profile roadmap --region ap-south-1 --filters 'Name=tag:Project,Values=aws-roadmap' --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name}' --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Instance inspection failed.' }
```

An empty result can be correct. A server-side `--filters` condition narrows service results; `--query` reshapes returned data using JMESPath. Neither grants permissions. Use structured JSON for programs instead of parsing a human table by column spacing.

## 2. Practice pagination locally

Save as `pagination_lab.py`:

```python
PAGES = {
    None: {"items": ["a", "b"], "next": "page-2"},
    "page-2": {"items": ["c"], "next": "page-3"},
    "page-3": {"items": [], "next": None},
}


def collect():
    token = None
    seen_tokens = set()
    output = []
    while True:
        page = PAGES[token]
        output.extend(page.get("items", []))
        token = page.get("next")
        if token is None:
            return output
        if token in seen_tokens:
            raise RuntimeError("Repeated continuation token")
        seen_tokens.add(token)


assert collect() == ["a", "b", "c"]
print("PASS: collected all pages, including an empty final page")
```

Explain why taking only the first response misses `c`. A service may return an empty page with a continuation token, so termination should follow the documented token contract rather than a universal “empty list means done” assumption.

In AWS CLI, inspect automatic pagination and the difference between `--page-size`, `--max-items`, and `--no-paginate`. In Boto3, use a supported paginator rather than writing a custom token loop unnecessarily. Not every operation has identical token names or a paginator. [CLI pagination](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-pagination.html)

## 3. Read a known S3 prefix with a paginator

Use the complete paginated SDK example in [the CLI/SDK reference](../../29-cli-and-sdk-reference.md) after creating a tiny private lab bucket. Save the Python block as a script and invoke it with the known bucket, region, profile, and prefix. Give the identity prefix-scoped listing permission; object content access is a separate permission.

Expected: object count and byte total match the known test objects. Repeat with an empty permitted prefix. Then remove the relevant permission in a dedicated test role or use an intentionally ungranted prefix and confirm an authorization failure is surfaced rather than converted into an empty successful result.

Do not run a broad full-account inventory or huge object listing merely to test pagination. A small controlled fixture plus the paginator contract is sufficient for this learning exercise.

## 4. Handle credentials and native-command errors

Refresh expired sessions through their supported login flow. Inspect configuration precedence using `aws configure list`; environment credentials/settings can override a profile. Do not dump credential files or full environment values into logs.

PowerShell's handling of native program failures differs from Python exceptions. Check `$LASTEXITCODE` immediately after `aws`. Preserve the sanitized operation/error code/request ID and stop dependent mutations when an earlier command fails. A create failure followed by an unguarded delete can target the wrong assumed resource.

In SDK code, distinguish validation, authorization, throttling, network timeout, and service errors. Retry only appropriate transient failures with a bounded overall deadline. A timeout after a write may require querying operation state using a stable identifier rather than blindly creating a duplicate.

## 5. Observe asynchronous state transitions

Use an existing small lab resource whose service provides a waiter, such as the S3 bucket workflow. Read the waiter's success/failure conditions and timeout behavior. For a job without an appropriate waiter, poll its documented status with a maximum elapsed time and backoff, and treat terminal failure differently from “still running.”

Write the state diagram before coding: `requested → creating/running → succeeded` or `failed/cancelled`. List what outputs are usable in each state. A resource ID returned by a create API does not prove the resource is ready to serve traffic.

Deliberately query a known absent test resource and record its specific not-found error. Compare that with a denied request. You must not claim cleanup succeeded merely because every read now fails for some unknown reason.

## 6. Use files for structured input

Prepare JSON in a dedicated file, parse it locally, and pass it through the documented `file://` mechanism where supported. Verify the shape expected by the parameter: tag arrays and tag maps are not interchangeable. Use `fileb://` only for a documented binary input case.

```powershell
$LabConfig = @{ Environment = 'learning'; Project = 'aws-roadmap' }
$LabConfig | ConvertTo-Json | Set-Content -LiteralPath './lab-config.json' -Encoding ascii
Get-Content -LiteralPath './lab-config.json' | ConvertFrom-Json
```

Avoid nested quoting and Bash line-continuation syntax in PowerShell. Use argument arrays when a generated command needs to preserve spaces without constructing executable shell text.

## 7. Close the resource lifecycle

Review create/update/read/delete behavior, asynchronous waits, retained artifacts, and permissions as one workflow. Delete only exact resources from the ledger, in dependency order. Inspect data versions/backups/logs and confirm whether they are intentionally retained.

Record which checks were local, which were read-only AWS calls, and which modified the sandbox. Preserve sanitized evidence and compare later cost data with the estimate. Continue to the full [S3/SQS lab](03-private-s3-and-sqs-labs.md).

References: [CLI command reference](https://docs.aws.amazon.com/cli/latest/reference/), [Boto3 paginators](https://docs.aws.amazon.com/boto3/latest/guide/paginators.html), [Boto3 errors](https://docs.aws.amazon.com/boto3/latest/guide/error-handling.html).
