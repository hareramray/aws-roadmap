# 01 — Establish a secure, identifiable AWS lab environment

[Account tutorials](README.md) · [All tutorials](../README.md)

**Mode:** local setup plus authorized account configuration. **Result:** a verified principal/region, protected account access, a working CLI profile, spending visibility, and a resource ledger. This tutorial does not require creating an Organization or upgrading a plan automatically.

## 1. Choose the account and record constraints

Use an existing training sandbox when available. Otherwise choose a personal account only after reviewing its current plan, payment/credit terms, services, and account features. Record account ID privately, owner, region, expiration of sandbox access, allowed actions, and maximum experiment spend.

Inspect current account settings rather than copying an older Free Tier tutorial. Account experiences and credit conditions can differ by signup cohort, including the October 2026 changes. Organizations/Control Tower and advanced-feature transitions can change eligibility or earning opportunities. Treat them as deliberate later lessons, not hidden prerequisites for a first S3 exercise. [Current Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/)

## 2. Protect recovery and root access

For an account you administer, verify the root email/contact details, strong unique password, MFA, and recovery information. Do not create root access keys or use root for daily labs. If an organization centrally manages root access, follow that supported model and document the authorized recovery path.

Create or obtain an everyday identity through the environment's supported federation/temporary-credential workflow. In a managed organization this often means an IAM Identity Center account assignment and permission set. An Identity Center account instance has different capabilities from an organization instance; do not assume one can replace the other for workforce AWS account access.

Use roles for workloads and short-lived developer sessions where available. If a legacy environment requires a long-lived key, scope/protect it, establish rotation/removal, and avoid embedding it in source or shell history. [IAM security practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

## 3. Set a budget and verify the alert path

Choose a learning allowance in your own currency and estimate service charges in their published billing currency, accounting for applicable taxes/conversion where needed. Configure an AWS Budget notification below the amount you cannot tolerate. Verify the notification destination and ownership.

Document that ordinary budget alerts are not universal immediate spending caps. If your account offers project spend-limit controls, inspect their specific coverage and pause behavior separately. Record at least two cost checks: before the lab and after usage data has arrived.

Create an estimate for a tiny S3/queue exercise, then a separate estimate for an idle EC2/ALB/database stack. Notice storage, NAT, public addresses, endpoints, snapshots, model inference, and log retention that can remain billed outside active application use. [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html), [Pricing Calculator](https://calculator.aws/)

## 4. Install and identify the CLI

Install AWS CLI v2 through the official OS-specific instructions. Verify the binary's version and path rather than assuming the first `aws` on `PATH` is correct:

```powershell
aws --version
Get-Command aws | Select-Object Source
git --version
python --version
```

For an existing Identity Center assignment, use the provided start/issuer URL, SSO region, account, and role. Do not invent those values or replace them with the deployment region automatically.

```powershell
aws configure sso --profile roadmap
aws sso login --profile roadmap
aws configure list --profile roadmap
aws sts get-caller-identity --profile roadmap --region ap-south-1 --no-cli-pager
```

Use your selected workload region instead of the illustrative `ap-south-1`. Expected: your intended account and assumed role. This proves identity, not permission to create every service. If your account uses another supported credential method, configure that method rather than creating an Organization solely to follow the SSO command. [CLI installation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), [CLI SSO setup](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)

## 5. Add a reusable account check

Before a script performs mutations, compare the active account with a value you deliberately set. Save this as part of your lab's setup block, not a credential file:

```powershell
$AwsProfile = 'roadmap'
$AwsRegion = 'ap-south-1'
$ExpectedAccount = 'REPLACE_WITH_SANDBOX_ACCOUNT_ID'
if ($ExpectedAccount -notmatch '^\d{12}$') { throw 'Set the intended 12-digit account ID.' }
$RawIdentity = aws sts get-caller-identity --profile $AwsProfile --region $AwsRegion --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot establish AWS identity.' }
$Caller = $RawIdentity | ConvertFrom-Json
if ($Caller.Account -ne $ExpectedAccount) { throw 'Wrong AWS account; stopping.' }
```

Deliberately set an incorrect expected account and verify the script stops before any create command. Restore the intended ID afterward. Do not confuse a friendly profile name with a checked account boundary.

## 6. Prepare repository and resource records

Create a local learning directory with code, infrastructure, tests, fixtures, and documentation. Ignore `.env`, credentials, caches, state containing secrets, and local runtime artifacts as appropriate. Record runtime/package versions and lock dependencies for reproducibility. Install Docker/IaC/Kubernetes/AI tools only when the relevant tutorial requires them.

Start a ledger with `account`, `region`, `service`, `resource ID`, `purpose`, `owner`, `created`, `cleanup method`, `retained dependencies`, and `estimated idle cost`. Use consistent names/tags where supported. Tags help identify resources but do not automatically enforce deletion or spending limits.

CloudShell may supply convenient tools under your existing identity, but service API calls still need permissions and can cost money. Local emulators can test code paths without proving real IAM/networking/service semantics. Label the evidence source accurately.

## 7. Finish the setup and rehearse cleanup

Run the [private S3 and SQS tutorials](03-private-s3-and-sqs-labs.md) in the permitted sandbox. Record exact resources before creation and verify their removal afterward. If a script fails mid-way, inspect what was already created rather than assuming rollback.

Check current objects/versions, snapshots, volumes, databases, registries, log groups, gateways/endpoints, and other resources relevant to the specific lab. Do not delete shared resources found through a broad list command. Recheck cost data after its normal delay.

**Completion:** demonstrate correct and intentionally wrong account checks, functioning everyday access, a verified notification path, and one completed lab with no unexplained retained resources. Use [CLI/SDK lifecycle practice](02-cli-sdk-and-resource-lifecycle.md) next.
