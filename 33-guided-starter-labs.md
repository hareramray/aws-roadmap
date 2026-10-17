# 33 — Guided starter labs: private S3 and SQS redelivery

[Roadmap index](README.md) · Prerequisites: [Account/tools](02-account-and-tooling.md), [IAM](04-iam-and-identity.md)

These two small labs connect concepts to observable behavior. They use **PowerShell and AWS CLI v2**, synthetic data, one region, and newly created resources. They are written instructions, not a claim that they have been run in your account. Review current service/account eligibility and pricing before running them.

Use an authorized sandbox with permission to perform the listed actions. Do not grant broad administrator access just to bypass a denial. A restricted training account or the newer simplified account experience may require an instructor-provided role or additional enabled capabilities. If setup is unavailable, follow the reasoning exercise without deploying.

## Common setup: account guard and checked commands

Run this in a new PowerShell session. Replace the profile, region, and expected account ID with your actual sandbox values. The check deliberately stops if you leave the account placeholder unchanged.

```powershell
$ErrorActionPreference = 'Stop'
$AwsProfile = 'roadmap'
$AwsRegion = 'ap-south-1'
$ExpectedAccount = 'REPLACE_WITH_YOUR_12_DIGIT_SANDBOX_ACCOUNT_ID'

if ($ExpectedAccount -notmatch '^\d{12}$') {
    throw 'Set ExpectedAccount to the intended sandbox account ID before continuing.'
}

$AwsContext = @('--profile', $AwsProfile, '--region', $AwsRegion, '--no-cli-pager', '--output', 'json')

function Invoke-LabAws {
    param([Parameter(Mandatory = $true)][string[]]$CliArgs)
    $Result = & aws @AwsContext @CliArgs
    if ($LASTEXITCODE -ne 0) {
        throw 'AWS command failed. Inspect the error and resource state before continuing.'
    }
    return $Result
}

$Caller = (Invoke-LabAws -CliArgs @('sts', 'get-caller-identity')) | ConvertFrom-Json
if ($Caller.Account -ne $ExpectedAccount) {
    throw 'The active account does not match the intended sandbox.'
}
$Caller | Select-Object Account, Arn

$LabSuffix = [guid]::NewGuid().ToString('N').Substring(0, 12)
$LabDirectory = Join-Path $env:TEMP ('aws-roadmap-' + $LabSuffix)
New-Item -ItemType Directory -Path $LabDirectory | Out-Null
```

Confirm that the ARN is your intended everyday role. If authentication fails, refresh it through your configured method, such as `aws sso login --profile roadmap`, then restart the check. The helper explicitly checks native exit codes; it stops on a failed API command rather than blindly continuing.

Keep this session open: the resource names and queue URL used for cleanup are stored in variables. Record them in your lab ledger in case the terminal closes. The temporary directory contains only local synthetic files created below.

## Lab A: a private S3 object and verified round trip

### What this proves

You will create a private general-purpose bucket, write a small object, read it with your authenticated role, compare its bytes, and test anonymous access. You will then delete the one known object and bucket. This lab deliberately does not enable versioning; the cleanup steps would need to change for a versioned bucket.

**Permissions to arrange:** `s3:CreateBucket`, `s3:DeleteBucket`, `s3:PutBucketPublicAccessBlock`, `s3:GetBucketPublicAccessBlock`, `s3:PutBucketOwnershipControls`, `s3:GetBucketOwnershipControls`, `s3:GetEncryptionConfiguration`, `s3:ListBucket`, and object-level `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` for the lab resource. The explicit ownership flag on bucket creation requires the ownership-controls write permission. Bucket and object actions use different resource ARN shapes; use the service authorization reference to scope the policy. Do not alter organization guardrails or unrelated buckets.

**Cost dimensions:** a few object/API requests and a tiny amount of storage; actual charges depend on account offers, region, and service prices. No NAT gateway, EC2 instance, or custom KMS key is needed for this exercise.

### A1. Create a uniquely named bucket

General-purpose bucket names must be globally unique within the AWS partition. Use the generated name below and retry with a new suffix if the name happens to be unavailable. Do not reuse an existing bucket for this exercise.

```powershell
$LabBucket = 'aws-roadmap-' + $LabSuffix
$CreateBucketArgs = @('s3api', 'create-bucket', '--bucket', $LabBucket, '--object-ownership', 'BucketOwnerEnforced')
if ($AwsRegion -ne 'us-east-1') {
    $CreateBucketArgs += @('--create-bucket-configuration', ('LocationConstraint=' + $AwsRegion))
}
Invoke-LabAws -CliArgs $CreateBucketArgs
Invoke-LabAws -CliArgs @('s3api', 'wait', 'bucket-exists', '--bucket', $LabBucket)

Invoke-LabAws -CliArgs @('s3api', 'put-public-access-block', '--bucket', $LabBucket, '--public-access-block-configuration', 'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true')
Invoke-LabAws -CliArgs @('s3api', 'get-public-access-block', '--bucket', $LabBucket)
Invoke-LabAws -CliArgs @('s3api', 'get-bucket-ownership-controls', '--bucket', $LabBucket)
Invoke-LabAws -CliArgs @('s3api', 'get-bucket-encryption', '--bucket', $LabBucket)
```

The `us-east-1` branch omits `LocationConstraint` because that API case differs from other regions. Review all four public-access-block values, the enforced ownership setting, and the encryption response. The lab does not add a public policy or ACL. S3's default encryption does not replace access control.

If an organization requires a different encryption policy, adapt the lab through an approved configuration rather than disabling that requirement. Key permissions and cleanup must then include that configuration.

### A2. Upload and download the known object

```powershell
$SourcePath = Join-Path $LabDirectory 'hello.txt'
$DownloadPath = Join-Path $LabDirectory 'hello-downloaded.txt'
'AWS roadmap private object lab' | Set-Content -LiteralPath $SourcePath -Encoding ascii

Invoke-LabAws -CliArgs @('s3api', 'put-object', '--bucket', $LabBucket, '--key', 'training/hello.txt', '--body', $SourcePath)
Invoke-LabAws -CliArgs @('s3api', 'head-object', '--bucket', $LabBucket, '--key', 'training/hello.txt')
Invoke-LabAws -CliArgs @('s3api', 'get-object', '--bucket', $LabBucket, '--key', 'training/hello.txt', $DownloadPath)

$SourceHash = (Get-FileHash -LiteralPath $SourcePath -Algorithm SHA256).Hash
$DownloadHash = (Get-FileHash -LiteralPath $DownloadPath -Algorithm SHA256).Hash
if ($SourceHash -ne $DownloadHash) {
    throw 'Round-trip file hashes do not match.'
}
'Round-trip SHA-256 hashes match.'
```

Record the content length, encryption response, and matching local hashes. Do not assume an S3 ETag is always an MD5 hash; behavior depends on upload/encryption details. The explicit local comparison tests the bytes actually uploaded and retrieved.

### A3. Test private access

This is an expected-failure check, so it bypasses the helper that throws on any failed command:

```powershell
& aws @AwsContext s3api head-object --bucket $LabBucket --key 'training/hello.txt' --no-sign-request
$AnonymousExit = $LASTEXITCODE
if ($AnonymousExit -eq 0) {
    throw 'Anonymous access unexpectedly succeeded. Stop and inspect the bucket configuration.'
}
```

Expected result: an authorization failure such as HTTP 403. A timeout, DNS error, or misspelled bucket name is **not** proof that permissions are correct. Inspect the actual error, confirm the same exact bucket/key still succeeds through the authenticated request, and review the public-access-block settings. This exercise shows two principals receiving different results; it is not a complete security audit.

### A4. Clean up the exact created resources

Recheck identity if your session/profile changed. Review the bucket inventory before deleting; it should contain only the synthetic lab object. If unexpected data appears, stop and investigate ownership rather than using a recursive delete.

```powershell
Invoke-LabAws -CliArgs @('s3api', 'list-objects-v2', '--bucket', $LabBucket)
Invoke-LabAws -CliArgs @('s3api', 'delete-object', '--bucket', $LabBucket, '--key', 'training/hello.txt')
Invoke-LabAws -CliArgs @('s3api', 'delete-bucket', '--bucket', $LabBucket)
Invoke-LabAws -CliArgs @('s3api', 'wait', 'bucket-not-exists', '--bucket', $LabBucket)
```

`delete-bucket` should fail rather than silently delete other objects if the bucket is not empty. The waiter checks nonexistence according to its documented behavior; investigate access/credential errors separately. Confirm the bucket no longer appears in your authorized inventory. Record cleanup time and check cost visibility later.

**Explain before moving on:** Which permission allowed upload? Why did the unsigned request fail? What did encryption protect? Why would versioning change deletion? Which charges could have occurred even for a tiny test?

## Lab B: SQS visibility timeout and message deletion

### What this proves

You will send one synthetic message, receive it without deleting it, make it available again, receive it a second time, and then explicitly delete it. The exercise demonstrates why receiving a message and completing work are separate operations. It does not promise exactly-once processing.

**Permissions to arrange:** `sqs:CreateQueue`, `GetQueueAttributes`, `SendMessage`, `ReceiveMessage`, `ChangeMessageVisibility`, `DeleteMessage`, `DeleteQueue`, and `GetQueueUrl` for the lab. The exact resource constraints for creation and subsequent actions differ. No public queue policy is needed.

**Cost dimensions:** a bounded number of queue requests. This uses SSE-SQS rather than a customer-managed KMS key, avoiding a separate key-management exercise. Read current SQS pricing and your account restrictions before running it.

### B1. Create a dedicated queue and send one message

```powershell
$LabQueueName = 'aws-roadmap-' + $LabSuffix
$QueueResult = (Invoke-LabAws -CliArgs @('sqs', 'create-queue', '--queue-name', $LabQueueName, '--attributes', 'VisibilityTimeout=30,MessageRetentionPeriod=3600,SqsManagedSseEnabled=true')) | ConvertFrom-Json
$LabQueueUrl = $QueueResult.QueueUrl
if ([string]::IsNullOrWhiteSpace($LabQueueUrl)) { throw 'Queue URL missing.' }

Start-Sleep -Seconds 2
Invoke-LabAws -CliArgs @('sqs', 'get-queue-attributes', '--queue-url', $LabQueueUrl, '--attribute-names', 'QueueArn', 'VisibilityTimeout', 'MessageRetentionPeriod', 'SqsManagedSseEnabled')

$MessagePath = Join-Path $LabDirectory 'job.json'
@{ jobId = ('job-' + $LabSuffix); action = 'demo-only' } | ConvertTo-Json -Compress | Set-Content -LiteralPath $MessagePath -Encoding ascii
$Sent = (Invoke-LabAws -CliArgs @('sqs', 'send-message', '--queue-url', $LabQueueUrl, '--message-body', ('file://' + $MessagePath))) | ConvertFrom-Json
$Sent | Select-Object MessageId
```

The short wait allows the newly created queue to become usable. If a transient creation-propagation error persists, inspect the exact error and retry the relevant read after a bounded delay; do not create many duplicate queues. Check the returned attributes before proceeding.

### B2. Receive with bounded long polling

An empty response is possible; it does not prove the queue is empty forever. This helper tries at most three long polls and throws if it still has no message. It uses the current message-system-attribute parameter for the receive count.

```powershell
function Receive-LabMessage {
    param([Parameter(Mandatory = $true)][string]$QueueUrl)
    for ($Attempt = 1; $Attempt -le 3; $Attempt++) {
        $Received = (Invoke-LabAws -CliArgs @('sqs', 'receive-message', '--queue-url', $QueueUrl, '--max-number-of-messages', '1', '--wait-time-seconds', '10', '--visibility-timeout', '30', '--message-system-attribute-names', 'ApproximateReceiveCount')) | ConvertFrom-Json
        if ($null -ne $Received.Messages -and $Received.Messages.Count -gt 0) {
            return $Received.Messages[0]
        }
    }
    throw 'No message received after three polls; inspect queue URL, producer result, and permissions.'
}

$First = Receive-LabMessage -QueueUrl $LabQueueUrl
if ($First.MessageId -ne $Sent.MessageId) { throw 'Unexpected message in this dedicated queue.' }
$First | Select-Object MessageId, Body, Attributes
```

The receipt handle is specific to a receive operation. Do not treat it as a stable message ID or publish it unnecessarily. The message becomes temporarily hidden from ordinary competing receives, but SQS standard-queue delivery is not an exclusive-processing guarantee. Consumers must still tolerate duplicate delivery.

### B3. Simulate an unfinished worker and redelivery

Instead of waiting for the full timeout, explicitly return the received message to availability. This models a worker that has not acknowledged successful completion. Run these steps before the current visibility window expires; if it has expired, receive the message again and use the latest handle.

```powershell
Invoke-LabAws -CliArgs @('sqs', 'change-message-visibility', '--queue-url', $LabQueueUrl, '--receipt-handle', $First.ReceiptHandle, '--visibility-timeout', '0')
$Second = Receive-LabMessage -QueueUrl $LabQueueUrl
if ($Second.MessageId -ne $First.MessageId) { throw 'Expected the same synthetic message.' }
$Second | Select-Object MessageId, Body, Attributes
Invoke-LabAws -CliArgs @('sqs', 'delete-message', '--queue-url', $LabQueueUrl, '--receipt-handle', $Second.ReceiptHandle)
```

Compare the receive count and message ID between reads. The stable business `jobId` is what an application could use to deduplicate work; the receipt handle is what this receive uses for deletion. Deleting a message is an acknowledgement to SQS, not evidence that your business action was durably committed.

Reason through this failure: the worker writes an output, crashes before deleting the message, and later receives it again. Explain how a conditional database write or a durable operation record could prevent a duplicate business result. An in-memory set would not survive replacement of the worker.

### B4. Clean up and verify

```powershell
Invoke-LabAws -CliArgs @('sqs', 'delete-queue', '--queue-url', $LabQueueUrl)
```

Queue deletion is asynchronous and can take time. After up to 60 seconds, check the dedicated queue by name:

```powershell
& aws @AwsContext sqs get-queue-url --queue-name $LabQueueName
$QueueLookupExit = $LASTEXITCODE
```

Expected result after deletion completes: the specific nonexistent-queue error. A credential, network, or access-denied error does not prove deletion. Do not treat an immediate lookup success during the deletion interval as a reason to recreate the queue. Confirm its absence in the intended account/region inventory.

The lab does not configure a DLQ. Continue with [serverless and integration](10-serverless-and-integration.md) to add bounded retries, a DLQ, idempotency, metrics, and safe redrive.

## Local artifacts and completion report

The temporary directory path is in `$LabDirectory`. It contains the synthetic source/download/message files created here, not cloud resources. Keep them for evidence if useful; otherwise remove only these exact files and then the empty directory:

```powershell
foreach ($LabFileName in @('hello.txt', 'hello-downloaded.txt', 'job.json')) {
    $LabFilePath = Join-Path $LabDirectory $LabFileName
    if (Test-Path -LiteralPath $LabFilePath -PathType Leaf) {
        Remove-Item -LiteralPath $LabFilePath
    }
}
if (@(Get-ChildItem -LiteralPath $LabDirectory -Force).Count -eq 0) {
    Remove-Item -LiteralPath $LabDirectory
}
```

Record which checks passed, the exact failures observed, what was deleted, and any intentionally retained artifact. If either lab stopped partway through, inspect the resource ledger and complete the appropriate exact-resource cleanup; do not assume a failed script rolled back previously successful API calls.

- [ ] I verified account, role, region, and required permissions.
- [ ] S3 authenticated round trip succeeded and file hashes matched.
- [ ] The unsigned object check failed for the expected authorization reason.
- [ ] The lab bucket and its known object were removed.
- [ ] The same SQS message was received again after becoming visible.
- [ ] I understand message ID, business idempotency key, and receipt handle.
- [ ] The dedicated queue was deleted and its absence verified.
- [ ] I checked remaining resources and later billing data.

## Official references

- [S3 create-bucket CLI reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html).
- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html).
- [S3 default encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-encryption.html).
- [SQS create-queue CLI reference](https://docs.aws.amazon.com/cli/latest/reference/sqs/create-queue.html).
- [SQS receive-message CLI reference](https://docs.aws.amazon.com/cli/latest/reference/sqs/receive-message.html).
- [SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html).
- [SQS delete-queue CLI reference](https://docs.aws.amazon.com/cli/latest/reference/sqs/delete-queue.html).
