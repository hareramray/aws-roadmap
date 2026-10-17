# Tutorial 03.1 — Map resource scopes and responsibility

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../03-cloud-foundations.md)

**Mode:** read-only AWS inspection plus a local design exercise. **Prerequisites:** an authorized sandbox session from [account setup](../../02-account-and-tooling.md), AWS CLI v2, and PowerShell. **Result:** a scope map and ownership matrix. Reviewed 2026-10-18.

## 1. Establish the exact account and Region

Create a local notes file with columns `resource`, `account`, `Region`, `AZ`, `network`, `owner`, and `evidence`. Use fabricated identifiers in public copies. Set your existing profile and selected Region; the example Region is illustrative:

```powershell
$labProfile = 'roadmap'
$labRegion = 'ap-south-1'
aws sts get-caller-identity --profile $labProfile --region $labRegion
aws ec2 describe-availability-zones --profile $labProfile --region $labRegion --query "AvailabilityZones[].{Name:ZoneName,Id:ZoneId,State:State}" --output table
aws ec2 describe-vpcs --profile $labProfile --region $labRegion --query "Vpcs[].{Id:VpcId,Cidr:CidrBlock,Default:IsDefault}" --output table
```

Check each command's exit status before interpreting output. The identity response should contain the intended account and your approved role/session ARN. Stop if it identifies root or another account. An access-denied result means the current role cannot perform that inspection; it does not establish that the service or resource is absent.

Record one AZ name and its AZ ID. The name is an account-facing label; AZ IDs are the appropriate identifiers for coordinating physical zone placement across accounts. Do not infer a universal mapping from the letter suffix.

## 2. Classify eight resources

Use this synthetic application even if your sandbox contains no resources:

| Resource | Correct scope | Why it matters |
| --- | --- | --- |
| Workforce role | Account identity | Another Region does not give it another independent identity |
| VPC | Region | It can contain subnets in multiple AZs |
| Subnet | One AZ | It cannot straddle two zones |
| EC2 instance | One AZ | Replaceable compute needs deliberate cross-AZ placement |
| EBS volume | One AZ | A restored volume can be created in another AZ; the original cannot simply attach there |
| General purpose S3 bucket | Selected Region | Regional storage is not automatically a second-Region recovery copy |
| Application Load Balancer | Region with configured zone placement | Its targets, health and enabled zones still need configuration |
| CloudFront distribution | Edge delivery service | Edge caching does not create a second application database |

Draw an account box around the Regional resources, a Region box around the VPC, and separate AZ boxes around each subnet. Put the bucket alongside the VPC, not inside a subnet: access through a VPC endpoint does not move S3 storage into your VPC.

**Negative check:** someone draws one subnet across both AZ boxes. Correct it by creating two conceptual subnets. Someone else draws the IAM role inside an EC2 box; correct that by showing a temporary session used by the workload instead of an embedded permanent credential.

## 3. Compare locality options

Add three fictional requirements: ordinary web users in India; a factory that must keep running during a WAN interruption; a latency-sensitive rendering client near a supported metropolitan area.

For the first, evaluate a normal Region using measured latency and service support. For the factory, examine local processing and an Outposts or other hybrid design only if operational and procurement constraints justify it. For the rendering client, investigate a supported Local Zone and its parent-Region dependencies. Edge locations support delivery functions; they are not interchangeable with Local Zones or customer-site infrastructure.

Your expected output is three decisions with assumptions, not three deployments. Write down connectivity, support, capacity, data location, and failure dependencies for each. A requirement for local execution does not automatically require a particular AWS hardware product.

## 4. Separate administrative and serving behavior

Trace two operations: `create a load balancer` and `send an HTTP request through an existing load balancer`. The first changes configuration through a control plane. The second uses the serving data plane. During an administrative API impairment, a mature design should not depend on creating its first recovery resource before serving any request.

Add a second failure: the deployment system cannot obtain a new secret because its dependency is unavailable. Existing application instances may continue serving with valid cached configuration, while replacement instances fail to initialize. Record the dependency explicitly; “Auto Scaling enabled” is not sufficient evidence of recovery.

## 5. Assign shared responsibilities

Fill this matrix for EC2-hosted versus managed-database implementations:

| Task | EC2-hosted application/database | Managed database choice |
| --- | --- | --- |
| Physical facility and underlying AWS host | AWS | AWS |
| Guest OS patching on your EC2 host | Your platform owner | AWS manages the service host; you still choose supported engine/configuration paths |
| SQL schema and inefficient queries | Your application/database owner | Your application/database owner |
| Runtime permissions and secrets | Your application/security owners | Your application/security owners |
| Recovery objectives and restore validation | Your service owner | Your service owner |

Add owners for vulnerability handling, key administration, logging, incident response, backups, and costs. Assign a named fictional role, such as `platform-oncall`, rather than “everyone.” Ask who acts when a managed-service maintenance event requires an application change.

## 6. Account boundaries and quotas

Draft a production/nonproduction/log-archive/security account layout for an established company. Explain why the management account should not host ordinary applications. Then produce a simpler personal-sandbox version. Do not create Organizations for this tutorial: account-plan and credit consequences belong in the setup decision.

Inspect one relevant quota in the Service Quotas console without requesting an increase. Record whether it is Regional, whether it is adjustable, and what symptom occurs when reached. Distinguish quota from capacity: permission to launch a number of instances does not reserve hardware in your chosen AZ.

## Verify, troubleshoot, and finish

You pass when every resource has a scope and owner, both traffic and administrative paths are visible, and an AZ failure differs from a Region failure in your explanation. If CLI output disagrees with the console, compare profile, account and Region first. If a list is empty, confirm filters and permission coverage before concluding there are no resources.

**Cleanup/cost:** these commands create no resources. Keep the local matrix and delete only disposable notes. Do not enable paid inventory, analysis or network services simply to complete the exercise.

Sources: [global infrastructure](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/global-infrastructure.html), [shared responsibility](https://aws.amazon.com/compliance/shared-responsibility-model/), [fault isolation boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/abstract-and-introduction.html).
