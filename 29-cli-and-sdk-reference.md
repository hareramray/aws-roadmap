# 29 — AWS CLI and SDK field reference

[Roadmap index](README.md) · Prerequisites: [Account/tools](02-account-and-tooling.md), [IAM](04-iam-and-identity.md)

Examples use **PowerShell and AWS CLI v2** because this workspace is on Windows. Replace resource placeholders with your own authorized sandbox identifiers. Inspect commands before running them. These examples have been reviewed as documentation; they have not been executed against your account.

## 1. Establish context before every session

```powershell
$AwsProfile = 'roadmap'
$AwsRegion = 'ap-south-1'
aws --version
aws configure list --profile $AwsProfile
aws sts get-caller-identity --profile $AwsProfile --region $AwsRegion --no-cli-pager
```

The region is an example; use the one selected in your plan. Confirm account and role before a write. If your profile uses IAM Identity Center and the session expired, run `aws sso login --profile roadmap`. Other supported credential methods have their own refresh workflows.

For a script with multiple native commands, check `$LASTEXITCODE` immediately after each command. PowerShell's `$ErrorActionPreference = 'Stop'` alone does not reliably turn every nonzero native program exit into a terminating exception across PowerShell versions.

```powershell
$IdentityJson = aws sts get-caller-identity --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Identity check failed; stop before creating resources.' }
$Identity = $IdentityJson | ConvertFrom-Json
$Identity | Select-Object Account, Arn
```

Avoid printing environment-variable values or credential files to diagnose authentication. `aws configure list` identifies configuration sources while masking credential values. CLI `--debug` output can contain sensitive request context; sanitize it before sharing.

## 2. Output, filters, and pagination

AWS CLI `--query` uses JMESPath to shape returned data. `--filters` is service-specific server-side filtering, while `--query` normally acts on results locally. Server-side filtering can reduce the response; a query does not change IAM permissions or make inaccessible resources visible.

```powershell
aws ec2 describe-instances --profile $AwsProfile --region $AwsRegion --filters 'Name=tag:Project,Values=aws-roadmap' --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name,Type:InstanceType,PrivateIP:PrivateIpAddress}' --output table --no-cli-pager
```

The CLI automatically paginates many list operations. `--page-size` controls service-page behavior; `--max-items` limits the CLI's aggregate output and can produce a continuation token; `--no-paginate` may leave you with only the first page. Do not mistake a partial list for a complete inventory. Check the operation's reference because pagination behavior and output queries can interact.

Prefer JSON for scripts and table/text for inspection. Parse JSON with `ConvertFrom-Json`; do not depend on table column spacing. Empty lists, absent fields, and null values are normal cases to handle.

## 3. Network and compute inspection

```powershell
aws ec2 describe-vpcs --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
aws ec2 describe-subnets --profile $AwsProfile --region $AwsRegion --query 'Subnets[].{Id:SubnetId,Vpc:VpcId,AZ:AvailabilityZone,CIDR:CidrBlock,FreeIPs:AvailableIpAddressCount}' --output table --no-cli-pager
aws ec2 describe-route-tables --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
aws ec2 describe-security-groups --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
aws ec2 describe-network-acls --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
aws ec2 describe-instance-status --include-all-instances --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
```

In a larger account, narrow results using known VPC/resource IDs and supported filters. Check route-table associations rather than assuming every subnet uses the main route table. An EC2 status check is not an application health check.

```powershell
$TargetGroupArn = '<your-target-group-arn>'
aws elbv2 describe-target-health --target-group-arn $TargetGroupArn --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
Resolve-DnsName '<your-hostname>'
Test-NetConnection -ComputerName '<your-hostname>' -Port 443
```

Target health can explain routing decisions, but test the application from the intended client network too. Do not open security groups broadly just to make a local laptop reach an intentionally private endpoint.

## 4. Storage and database inspection

```powershell
$BucketName = '<your-sandbox-bucket>'
aws s3api get-public-access-block --bucket $BucketName --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws s3api get-bucket-encryption --bucket $BucketName --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws s3api get-bucket-versioning --bucket $BucketName --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws s3api list-objects-v2 --bucket $BucketName --prefix 'training/' --profile $AwsProfile --region $AwsRegion --query 'Contents[].{Key:Key,Bytes:Size,Modified:LastModified}' --output table --no-cli-pager
aws rds describe-db-instances --profile $AwsProfile --region $AwsRegion --query 'DBInstances[].{Id:DBInstanceIdentifier,Engine:Engine,Status:DBInstanceStatus,MultiAZ:MultiAZ,Public:PubliclyAccessible}' --output table --no-cli-pager
aws dynamodb describe-table --table-name '<your-table>' --profile $AwsProfile --region $AwsRegion --no-cli-pager
```

Listing objects can expose sensitive names; limit scope and share only sanitized output. A missing optional bucket configuration can produce a specific not-found error; distinguish that from an inaccessible bucket or failed credential. Do not infer all security properties from a single flag.

## 5. Application and operational inspection

```powershell
aws lambda list-functions --profile $AwsProfile --region $AwsRegion --query 'Functions[].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,Timeout:Timeout}' --output table --no-cli-pager
aws ecs list-clusters --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws eks list-clusters --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws cloudformation list-stacks --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws cloudwatch describe-alarms --state-value ALARM --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBucket --profile $AwsProfile --region $AwsRegion --no-cli-pager
```

CloudTrail event history has specific scope, retention, and supported event categories; it is not a query across every audit event ever generated. A missing event does not prove no action occurred. Check the configured trail/data-event coverage and the relevant region/time window.

For one known log group, inspect a short interval:

```powershell
aws logs tail '<your-log-group>' --since 10m --profile $AwsProfile --region $AwsRegion --no-cli-pager
```

Log queries and reads can incur charges and reveal application data. Start with a narrow time window, a known group, and an explicit investigation question. See [operations](16-observability-and-operations.md).

## 6. Quotas and available configuration

```powershell
aws service-quotas list-service-quotas --service-code ec2 --profile $AwsProfile --region $AwsRegion --no-cli-pager
aws ec2 describe-availability-zones --profile $AwsProfile --region $AwsRegion --query 'AvailabilityZones[].{Name:ZoneName,Id:ZoneId,State:State}' --output table --no-cli-pager
```

Not every quota is exposed through the same API, and not every quota is adjustable. A sufficient quota does not guarantee instantaneous capacity for every instance type/AZ. Consult the exact service guide for the intended configuration.

## 7. JSON files and PowerShell quoting

For complex AWS inputs, prefer a JSON file over nested shell quoting. Write text explicitly with an appropriate encoding and review it before submission. The following generates a local example file only:

```powershell
$Example = @{ Project = 'aws-roadmap'; Environment = 'learning' }
$Example | ConvertTo-Json | Set-Content -LiteralPath './example-tags.json' -Encoding ascii
Get-Content -LiteralPath './example-tags.json'
```

Different APIs expect different shapes: tag maps and arrays of `{Key,Value}` are not interchangeable. Use `file://path.json` only for parameters that accept that JSON shape. Binary payloads may require `fileb://` or explicit CLI binary-format handling; consult the operation reference.

Avoid Bash backslash continuations in PowerShell. The examples here keep commands on one line to reduce quoting and trailing-whitespace mistakes. If a command contains a placeholder, replace it before running rather than letting the API interpret the placeholder as a real name.

## 8. SDK example: paginated, read-only S3 listing

Install a supported `boto3` version in a virtual environment and lock dependencies for a real project. This example uses the same named profile as the CLI and reads only keys under a chosen prefix. Give the role `s3:ListBucket` permission scoped to the bucket and prefix as appropriate; it does not need object-read permission merely to list keys.

```python
import argparse
import logging

import boto3
from botocore.config import Config
from botocore.exceptions import BotoCoreError, ClientError


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--bucket", required=True)
    parser.add_argument("--profile", default="roadmap")
    parser.add_argument("--region", required=True)
    parser.add_argument("--prefix", default="training/")
    args = parser.parse_args()

    client = None
    count = 0
    total_bytes = 0
    try:
        session = boto3.Session(profile_name=args.profile, region_name=args.region)
        client = session.client(
            "s3",
            config=Config(
                connect_timeout=3,
                read_timeout=10,
                retries={"mode": "standard", "total_max_attempts": 4},
            ),
        )
        pages = client.get_paginator("list_objects_v2").paginate(
            Bucket=args.bucket,
            Prefix=args.prefix,
        )
        for page in pages:
            for obj in page.get("Contents", []):
                count += 1
                total_bytes += obj["Size"]
        print({"objects": count, "bytes": total_bytes})
    except ClientError as exc:
        response = exc.response
        logging.error(
            "AWS error=%s request_id=%s",
            response.get("Error", {}).get("Code", "unknown"),
            response.get("ResponseMetadata", {}).get("RequestId", "unknown"),
        )
        raise SystemExit(1)
    except BotoCoreError:
        logging.error("SDK or credential failure; inspect local configuration securely")
        raise SystemExit(1)
    finally:
        if client is not None:
            client.close()


if __name__ == "__main__":
    main()
```

The timeout/retry values are educational choices, not universal production settings. A broad prefix can still cause many requests. Add cancellation, work bounds, telemetry, and application-specific error handling as requirements grow. In a deployed workload, use its execution role and the normal credential provider chain instead of requiring a developer profile.

## 9. Automation habits

Use paginators for list APIs, waiters or bounded polling for asynchronous state changes, and idempotency tokens/conditional operations where supported. Retry transient errors within an overall deadline; do not retry authorization or validation failures forever. Record request IDs and resource identifiers without logging credentials or sensitive payloads.

Review differences between CLI and SDK behavior, including pagination defaults and exception types. Never run a delete loop over an unreviewed inventory. Capture the intended resource set, verify ownership/account/region, and use dependency-aware teardown.

## Official references

- [AWS CLI reference](https://docs.aws.amazon.com/cli/latest/reference/).
- [CLI filtering](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html).
- [CLI pagination](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-pagination.html).
- [CLI quoting](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-parameters-quoting-strings.html).
- [Boto3 credentials](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html).
- [Boto3 paginators](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/paginators.html).
- [Botocore configuration](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html).
