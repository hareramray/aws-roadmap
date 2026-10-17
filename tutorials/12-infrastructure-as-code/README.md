# Infrastructure as code tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../12-infrastructure-as-code.md)

Use [CloudFormation, CDK, and SAM](01-cloudformation-cdk-sam.md) to inspect the AWS-managed stack lifecycle. Use [Terraform/OpenTofu state](02-terraform-opentofu-state.md) to learn provider configuration, plans, locking, and recovery. Choose one owner per resource; do not run both tutorials against the same queue.

| Roadmap topics | Tutorial evidence |
| --- | --- |
| Desired versus observed state, dependency graph, lifecycle, replacement, deletion, drift | Tutorial 1 creates an encrypted queue, previews an update, detects deliberate drift, and restores desired state. Tutorial 2 compares configuration, state, and remote objects. |
| CloudFormation parameters, conditions, intrinsic functions, outputs, references, change sets, events, rollback | Tutorial 1 provides a complete template and sequential commands with checked exit codes, plus failure diagnosis and a condition extension. |
| Retain/Snapshot/Delete, update replacement, stack policy, termination protection, nested stacks, StackSets | Tutorial 1 explains each protection's scope and uses an isolated import/retain recovery drill rather than treating deletion as backup. |
| CDK constructs, synthesis, context, bootstrap roles/assets, SAM transform/build/deploy/local testing | Tutorial 1 supplies a CDK application and a SAM function, inspects generated CloudFormation, and keeps all three resource owners separate. |
| Terraform/OpenTofu providers, variables, outputs, modules, iteration, references, version pinning and dependency locks | Tutorial 2 provides typed HCL, a reviewed plan workflow, and a stable `for_each` module extension. |
| State secrets, S3 remote backend, versioning, encryption, native locking, IAM, workspaces, recovery | Tutorial 2 migrates a dedicated state object, explains lock permissions, and specifies a controlled lock/recovery drill. |
| Imports, moved resources, drift refresh, concurrent edits, force-unlock risk | Both tutorials inspect existing objects before ownership changes; tutorial 2 uses a configuration-driven import and a `moved` block. |
| CI identity, linting, policy checks, dependency graphs, environment boundaries, plan review, drift schedules | Tutorial 2 exports a machine-readable plan for checks and links to the [secure delivery pipeline](../13-devops-and-delivery/01-secure-build-and-release-pipeline.md). |
| Cost, quotas, resource inventory, cleanup | Both tutorials distinguish local validation from account operations and verify retained resources after deletion. |

Tool binaries and dependencies must already be installed using their current official instructions. Pin a reviewed CLI/provider/CDK version in your project and retain its lockfile. The tutorials do not ask you to download unsigned binaries or expose long-lived credentials. Templates and plans are review artifacts, not permission to change an arbitrary account.
