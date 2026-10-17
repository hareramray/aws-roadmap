# 02 — Account setup, cost boundaries, and developer tooling

[Roadmap index](README.md) · Prerequisite: [Technical basics](01-prerequisites.md)

**Outcome:** an identifiable, secured learning environment with billing visibility, temporary credentials where available, and a repeatable start/finish routine. This chapter describes setup; it does not create an AWS account or deploy resources.

## 1. Choose your learning environment deliberately

You can use an employer-provided training account, a provisioned learning sandbox, or a personal AWS account. Use only environments in which you are authorized to experiment. Separate experiments from production and never test recovery by deleting business data.

For a personal account, use an email address you control long term, accurate billing/contact details, strong unique credentials, and MFA on the root user. Store recovery information securely. Do not create root access keys, and do not use root for routine learning. Some account-level tasks still require root or a supported centralized root-access workflow; document how you would regain access.

Record account alias, account ID privately, selected region, environment purpose, owner, and permitted activities. A name such as `learning` is helpful, but it does not prove which account your CLI is using.

## 2. Understand the account plan before building

As reviewed on 18 October 2026, AWS advertises a Free account plan for eligible new customers and a Paid account plan, with differing service access and billing behavior. The Free plan ends after six months or when credits are exhausted, whichever happens first; older accounts can have different eligibility. Check the exact plan and credit expiry shown in your account. Do not assume an old “12 months free” tutorial applies. [AWS Free Tier](https://aws.amazon.com/free/)

The October 2026 signup experience introduces account/project behavior that differs from older accounts. Joining an AWS Organization or setting up a Control Tower landing zone ends credit-earning activities and eligibility to earn more credits under the current FAQ. Verify the treatment and expiry of already-earned credits for your account cohort before enabling multi-account tooling. Do not infer those terms from an older search-result excerpt. [Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/)

Some granular IAM, multi-region, and enterprise workflows may require activating advanced features in the newer account experience. Review that transition and its plan implications before following later chapters. The account settings and current signup documentation determine which setup path applies to you; this roadmap's full IAM/CLI labs assume a sandbox with the required capabilities enabled.

A chapter using a restricted service may require a different sandbox or plan. Do not upgrade reflexively to make a tutorial match. A local exercise or architecture-only exercise can teach the same early concept without a larger account commitment.

## 3. Establish everyday access

Prefer federation and temporary role credentials. In an existing organization, ask for a sandbox assignment through IAM Identity Center or the organization's approved identity provider. For a personal account, evaluate whether an organization-based IAM Identity Center setup fits your plan and learning needs before enabling it. An account instance of Identity Center has different capabilities from an organization instance; do not assume it provides the same multi-account AWS access workflow.

If your environment has no federation configured, use its supported short-lived console/CLI sign-in workflow or a carefully scoped non-root identity while planning a better long-term setup. Avoid making long-lived IAM user keys the default for every lab. Where a legacy workflow requires them, protect, narrowly scope, rotate, and remove them when no longer needed. Never place credentials in Markdown, source code, chat, container images, or shell commands committed to Git.

Separate day-to-day permissions from administrative setup. A learner does not need unrestricted production access. Roles for application workloads are distinct from roles you use as a human. Review [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) and [the identity chapter](04-iam-and-identity.md).

## 4. Make spending visible

Set a personal monthly learning allowance and translate it into a smaller per-lab budget. Configure an AWS Budget with notifications at thresholds below your maximum tolerance; confirm delivery of the notification. Review billing and available cost views before and after experiments. Cost data and alerts are not necessarily immediate, and a budget alert is not a universal spending cap. [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)

The newer account/project experience also offers specific spend-limit controls. Their eligibility, scope, and pausing behavior are separate from ordinary AWS Budgets; inspect the controls available in your account and understand what they cover. [Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/)

For every lab, list cost dimensions: runtime hours, storage, requests, data transfer, retrieval, provisioned throughput, licensed software, and optional telemetry. Small resources can still incur costs through NAT gateways, public IPv4 addresses, load balancers, interface endpoints, cross-region replication, retained snapshots, managed cluster control planes, and model inference.

Use [AWS Pricing Calculator](https://calculator.aws/) with your intended region and configuration. Include “idle overnight” and “forgotten for a month” scenarios. Free usage allowances, credits, and stopping an instance do not eliminate every attached charge.

## 5. Choose a region and naming scheme

Start with one region that supports the services you need and meets any data-location requirements. Compare latency, prices, quotas, service availability, and organizational restrictions. The closest city is not always the appropriate region. Some services are global or have control-plane operations in particular regions; follow service documentation.

Use a naming convention such as `roadmap-dev-api` and tags such as `Project=aws-roadmap`, `Environment=learning`, `Owner=<your-id>`, and `ExpiresOn=<date>`. Tags help inventory and cost analysis, but they do not automatically delete resources or enforce permissions. Some resources do not support the same tagging capabilities.

Keep a resource ledger: account, region, service, identifier, purpose, creation date, cleanup method, and retained dependencies. Inventory all regions used by your exercise, not just the region currently selected in the console.

## 6. Install and verify tools

Install AWS CLI v2 from the official package for your operating system. Verify the executable with `aws --version`; do not assume the `aws` on your path is the expected installation. Install Git, an editor, a current supported Python or Node runtime, and optionally Docker. Add Terraform/OpenTofu, CDK, SAM, or Kubernetes tools only when the relevant chapter needs them. [AWS CLI installation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

For a supplied IAM Identity Center account, configure a named profile using the start/issuer URL and SSO region provided by your administrator. The SSO region can differ from the region in which you deploy workloads. [CLI Identity Center configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)

```powershell
aws --version
aws configure sso --profile roadmap
aws sso login --profile roadmap
aws sts get-caller-identity --profile roadmap --region ap-south-1 --no-cli-pager
aws configure list --profile roadmap
```

`ap-south-1` is an example, not a required region. Verify the returned account and ARN before any mutation. `get-caller-identity` identifies your principal; it does not demonstrate authorization to create a service resource. Do not publish its output if account identifiers or role names are private.

The CLI configuration lives under your user profile's `.aws` directory by default. Cached credentials and SSO tokens are sensitive even when they expire. Environment variables can override profile configuration; use `aws configure list` to diagnose the source of settings without dumping secrets.

## 7. CloudShell, local environments, and emulators

CloudShell can be convenient for an authenticated shell near the console, subject to service availability and supported features. It does not make AWS API operations free or grant additional permissions. Protect files and credentials there as you would locally.

Local emulators and mocks can accelerate application testing, but they do not prove AWS IAM evaluation, regional availability, networking, managed-service quotas, or all service semantics. Separate local unit tests from a small authorized integration test against actual AWS resources.

Keep a project directory containing application code, infrastructure definitions, documentation, tests, and sanitized sample data. Use sample values, environment variables, or parameter stores for configuration; use an approved secret store for real secrets. Never commit state files or outputs without reviewing them for sensitive information.

## 8. First-session verification

1. Sign in using your everyday role and check the account/region.
2. Confirm root MFA and account recovery details through the appropriate setup process.
3. Confirm your plan, service access, billing contacts, budget notifications, and learning allowance.
4. Run the identity check above and record the result privately.
5. Read a service resource list with your role; understand an access denial if permissions are intentionally narrow.
6. Create a lab journal and resource ledger using [the templates](32-engineering-templates.md).
7. Complete [the private S3 starter lab](33-guided-starter-labs.md) and verify its teardown.

## 9. End-of-session routine

Remove resources through the same IaC workflow that created them when possible. Review retained objects/versions, EBS volumes, snapshots, database backups, log groups, elastic IP allocations, gateways, endpoints, registries, and subscriptions relevant to your lab. Emptying a versioned bucket requires considering historical versions and delete markers, not just current objects.

Do not delete shared infrastructure because it appears in a search result. Verify the account, ownership tags, resource ID, and dependency graph. Keep required audit or recovery evidence. Check cost views again after billing data has had time to arrive, and investigate residual charges rather than repeatedly deleting unrelated resources.

## Completion gate

- [ ] I can identify my account, region, role, and credential source.
- [ ] Root is protected and is not my daily identity.
- [ ] I understand the restrictions and billing behavior of my account plan.
- [ ] Notifications are configured, and I know they are not a universal hard cap.
- [ ] I can estimate a lab and identify its persistent cost drivers.
- [ ] My tools work without embedded credentials.
- [ ] I have a resource ledger and a repeatable cleanup procedure.
