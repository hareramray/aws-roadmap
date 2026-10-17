# Plan, lock, and recover Terraform or OpenTofu state

[IaC tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../12-infrastructure-as-code.md)

**Outcome:** Manage one encrypted queue with a reviewed saved plan, migrate its state to a protected S3 backend, and practice ownership changes without recreating the resource. Choose **one** reviewed Terraform or OpenTofu version for this exercise. Their releases, providers, lock behavior, and state compatibility must be checked independently; do not alternate binaries against the same state casually.

Local editing/formatting creates no AWS resources. Provider initialization downloads software. Plan/refresh normally read AWS. Apply, backend creation, and destruction change your account and can incur charges. Use a dedicated directory and sandbox identity; never copy real credentials into HCL or backend files.

## 1. Pin the toolchain and write typed configuration

Record the exact CLI binary version/checksum and provider version approved for your platform. Create `main.tf` below. Replace `REVIEWED_EXACT_VERSION` with the reviewed AWS provider version before initialization; Terraform/OpenTofu do not accept that placeholder as a version constraint. Set `required_version` to an exact engine version for a production project after confirming native S3 lock support. The example range only establishes the feature-era minimum and excludes a future major version.

```hcl
terraform {
  required_version = ">= 1.10.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= REVIEWED_EXACT_VERSION"
    }
  }
}

variable "region" {
  type = string
}

variable "queue_name" {
  type = string
  validation {
    condition     = can(regex("^roadmap-[a-z0-9-]{3,50}$", var.queue_name))
    error_message = "Use a unique roadmap- prefixed sandbox queue name."
  }
}

provider "aws" {
  region = var.region
}

resource "aws_sqs_queue" "orders" {
  name                       = var.queue_name
  visibility_timeout_seconds = 60
  message_retention_seconds  = 345600
  sqs_managed_sse_enabled     = true
  tags = { Purpose = "roadmap-iac" }
}

output "queue_url" {
  value = aws_sqs_queue.orders.url
}
```

Use your standard AWS profile/default credential chain. Set environment variables rather than writing credentials:

```powershell
$LabRegion = if ($env:AWS_REGION) { $env:AWS_REGION } else { $env:AWS_DEFAULT_REGION }
if (-not $LabRegion) { throw 'Set AWS_REGION or AWS_DEFAULT_REGION to the reviewed sandbox Region' }
$env:TF_VAR_region = $LabRegion
$env:TF_VAR_queue_name = 'roadmap-' + [guid]::NewGuid().ToString('N')
$Iac = 'terraform' # Set to 'tofu' if that is your chosen, reviewed binary.
& $Iac version
if ($LASTEXITCODE -ne 0) { throw 'CLI unavailable' }
& $Iac fmt -check
if ($LASTEXITCODE -ne 0) { throw 'Format the configuration before proceeding' }
& $Iac init
if ($LASTEXITCODE -ne 0) { throw 'Initialization failed' }
& $Iac validate
if ($LASTEXITCODE -ne 0) { throw 'Validation failed' }
```

Commit configuration and the dependency lockfile. Exclude `.terraform/`, state/backup files, saved plans, credentials, and sensitive variable files from version control. A lockfile records provider selections/checksums; it is not a remote-state lock. A value marked `sensitive` can still exist in state or a plan.

## 2. Create and inspect a saved plan

Confirm caller identity first. Run `& $Iac plan -out=create.plan`, check the exit code, then `& $Iac show create.plan`. Expect exactly one queue addition. Review account/Region, name, encryption, and visibility. Apply that exact file with `& $Iac apply create.plan`; a saved plan can be applied without an additional interactive confirmation, so the preceding review is meaningful.

Run `& $Iac output -raw queue_url` and save the URL in the resource inventory. Verify queue attributes through the AWS CLI. Run another plan; expect no changes. If using `-detailed-exitcode`, remember 0 means no difference, 2 means a valid plan with changes, and 1 means failure; do not reject 2 as an error in CI.

Change visibility to 90, generate a new saved plan, inspect the in-place update, apply, and verify. Do not apply an old plan after unrelated state changes. Export `show -json` only into a protected local artifact because it may contain sensitive values; use it to check for unexpected deletes/replacements and overly broad policies in richer projects.

## 3. Create a protected backend independently

Use a separately provisioned dedicated S3 bucket with public access blocked, versioning enabled, encryption, and tightly restricted access. Backend bootstrap has its own lifecycle; do not make the application depend on destroying the bucket that holds its own state. A customer managed KMS key adds key policy/permissions/cost and must remain available for recovery.

Add `backend "s3" {}` inside the existing `terraform` block. Create a nonsecret `backend.hcl`, replacing all placeholders:

```hcl
bucket       = "YOUR_DEDICATED_STATE_BUCKET"
key          = "roadmap/queue/terraform.tfstate"
region       = "YOUR_REGION"
encrypt      = true
use_lockfile = true
```

Use a tool version that supports this configuration. Terraform documents native S3 locking and deprecates its DynamoDB locking option. OpenTofu also documents native S3 locking, while its DynamoDB support has a different lifecycle; do not copy one product's deprecation statement onto the other. Consult the documentation for the chosen pinned release.

Grant the runner only the required bucket listing and object read/write permissions for the state prefix and lock object. Native locking needs the documented `.tflock` object permissions, including deletion of the lock object; state-object deletion is a separate permission. Include KMS permissions if applicable. Keep account/environment states separate with distinct keys, access controls, and preferably distinct trust boundaries. CLI workspaces alone are not an IAM isolation boundary.

## 4. Migrate and verify ownership

Ensure no other process is applying this state. Make a secure local backup, record the current state lineage/serial and resource address, then run:

```powershell
& $Iac init -migrate-state -backend-config=backend.hcl
if ($LASTEXITCODE -ne 0) { throw 'State migration failed; preserve both copies for diagnosis' }
& $Iac state list
if ($LASTEXITCODE -ne 0) { throw 'State read failed' }
& $Iac plan
if ($LASTEXITCODE -ne 0) { throw 'Post-migration plan failed' }
```

Review the migration prompt; do not silently overwrite an existing destination state. Expect `aws_sqs_queue.orders`, no resource recreation, and a versioned state object in S3. Inspect bucket versions and ensure access is denied to an unrelated test identity. Do not publish state contents as evidence; record object version/lineage metadata securely.

For a lock drill, use a dedicated test copy of the configuration/state boundary and a controlled pending operation according to the chosen tool's documented behavior. A second writer should wait or fail with lock information. Never fabricate/delete a lock object in an active shared backend. If a real process crashes, verify it has stopped and no runner owns the operation before using the tool's documented force-unlock with the exact lock ID. Unlocking cannot repair corrupted state.

## 5. Refactor and import without recreating

Rename the resource label from `orders` to `accepted_orders`, update output references, and add:

```hcl
moved {
  from = aws_sqs_queue.orders
  to   = aws_sqs_queue.accepted_orders
}
```

Plan and verify an address move rather than destroy/create before applying. Keep move history while relevant configurations may still need migration. For several queues, use a map with stable business keys and `for_each`; list-index `count` can change addresses unexpectedly when items are reordered. Move the repeated queue definition into a small module with typed inputs/outputs only after the resource boundary is understood.

For import, create a different disposable unmanaged queue, define its matching resource configuration, and use a configuration-driven `import` block with that provider's documented queue URL ID. Preview before applying. Import places the object under management; it does not guarantee matching properties. If another stack owns it, resolve ownership first. Never import the tutorial's CloudFormation queue while CloudFormation still manages it.

## 6. Drift, recovery, teardown

Change this lab queue's visibility outside the tool and run a normal plan. Distinguish refreshing observed values from changing the desired configuration. A refresh-only plan can accept remote observations into state, but it does not change your HCL or necessarily resolve a policy violation. Review what outcome you actually want.

To recover a lost/corrupt state, stop writers, preserve evidence, identify a trusted versioned backup, compare it to current remote resources, and restore through the chosen tool's documented state workflow. Avoid hand-editing JSON. A historical state version may omit resources created later; blindly restoring it can orphan or duplicate infrastructure.

Generate `destroy.plan` using `plan -destroy -out=destroy.plan`, inspect that only lab-owned objects are deleted, then apply it. Verify the queue is gone and state has no remaining managed resources. Retain a protected final state backup per your policy. Delete the dedicated backend only after all users are finished and every object version/delete marker is accounted for; shared backends must remain. Remove local plans/state copies securely.

Explain how a provider lockfile differs from a state lock, why a saved plan is sensitive, why force-unlock is not recovery, and how a moved block avoids unintended replacement.

Official references: [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3), [OpenTofu S3 backend](https://opentofu.org/docs/language/settings/backends/s3/), [Terraform imports](https://developer.hashicorp.com/terraform/language/import), [resource refactoring](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring), [state purpose](https://developer.hashicorp.com/terraform/language/state/purpose).
