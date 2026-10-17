# 12. Infrastructure as code and configuration management

[Roadmap index](README.md)

**Prerequisites:** [tooling](02-account-and-tooling.md), [cloud foundations](03-cloud-foundations.md), [IAM](04-iam-and-identity.md), and the service modules for resources you will provision. Follow with [delivery pipelines](13-devops-and-delivery.md).

**Outcome:** Reproduce an environment from reviewed code, predict changes before applying them, protect infrastructure state, recover from failed updates, and remove a lab without deleting shared resources.

## 1. Learn the infrastructure lifecycle

Infrastructure as code (IaC) describes resources and their relationships in version-controlled files. A tool compares configuration, its recorded state, and actual infrastructure, then performs changes through provider APIs. Declarative configuration does not eliminate imperative effects: replacing a database, updating a policy, or changing a route can still disrupt a system.

Learn desired state, actual state, drift, dependency graphs, resource identity, imports, replacements, rollback, and eventual consistency. Distinguish creating infrastructure from configuring software inside it. Prefer immutable host images or containers when practical, while using configuration management for systems that require ongoing configuration.

Before any apply operation, answer: which account and Region, which state or stack, which identity, what additions/changes/deletions, what data is at risk, and how is success verified? A useful plan is a review artifact, not evidence that deployment has already succeeded.

Use one authoritative manager per resource. Do not let CloudFormation and Terraform independently manage the same security-group rule or bucket policy. Shared ownership produces drift and can cause one tool to undo another's changes.

## 2. Choose a tool deliberately

| Tool | Model | Strengths and responsibilities |
| --- | --- | --- |
| CloudFormation | AWS templates and service-managed stacks | Native AWS resource management; learn stack events, change sets, rollback, resource support, and retention semantics |
| AWS CDK | Programming-language constructs that synthesize CloudFormation | Reusable abstractions and normal language tooling; still requires CloudFormation knowledge and review of synthesized output |
| AWS SAM | Serverless-focused template transform and tooling | Efficient Lambda/API/event application workflows; understand generated resources and deployment behavior |
| Terraform | Provider-based declarative configuration and state | Broad ecosystem; own provider/version compatibility, state access, locking, and migration |
| OpenTofu | Open-source declarative configuration and state ecosystem | Similar foundational workflow; verify its own feature/version/backend documentation instead of assuming all Terraform behavior matches |
| Pulumi | General-purpose languages with an infrastructure engine | Language reuse and typed abstractions; understand its separate state, secrets, provider, and deployment model |
| Ansible / Systems Manager | Configuration and operational automation | Useful for host configuration and maintenance; not a substitute for knowing resource ownership and desired state |

Become productive with one primary tool and learn to read at least one other. For AWS-native serverless work, SAM or CDK is a practical starting point. For infrastructure teams already using a provider-based workflow, Terraform or OpenTofu can fit. Tool selection should account for team skills, lifecycle ownership, compliance requirements, and migration cost.

## 3. Master CloudFormation fundamentals

Study template format, resources, parameters, outputs, conditions, mappings, intrinsic functions, references, attributes, and dependencies. Prefer references that establish dependencies naturally; add explicit dependencies only when the relationship cannot be inferred. Understand logical IDs versus physical resource names. A seemingly harmless logical-ID rename can mean deletion and creation.

Inspect change sets for replacements and security changes. A change set describes expected resource changes but does not guarantee the update will succeed; permissions, quotas, dependencies, and runtime constraints can still cause failure. Read stack events to find the first meaningful failure rather than only the final rollback status. See [CloudFormation change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html).

Learn `DeletionPolicy`, `UpdateReplacePolicy`, termination protection, stack policies, and service-specific backup behavior. These controls solve different problems. `DeletionPolicy: Retain` can leave a resource billable after stack deletion; it does not put that resource into another managed stack. `UpdateReplacePolicy` governs the old physical resource during replacement. Document who owns retained resources and how they are eventually removed.

Use nested stacks or other composition carefully. Split by lifecycle and ownership, not an arbitrary resource count. A network shared by several applications should not disappear with one application's experiment. Cross-stack exports can create update dependencies; explicit contracts are useful, but tightly coupling every output makes change difficult.

Learn drift detection and its supported-resource limitations. Not every property or external change is detectable. Understand resource import before adopting existing infrastructure. For many accounts or Regions, study StackSets, delegated administration, operation concurrency, failure tolerance, and organizational targeting.

## 4. Learn CDK and SAM as abstraction layers

For CDK, understand apps, stacks, constructs, construct levels, synthesis, assets, context, bootstrapping, environment binding, and deployment roles. Bootstrap resources can include buckets, repositories, and roles; examine their trust relationships and lifecycle. Pin compatible CLI/library dependencies and commit the relevant dependency lockfiles.

Constructs save repetition, but defaults may create resources you did not budget for. Review the synthesized template and `cdk diff`, especially IAM permissions, network topology, encryption, removal policies, and data retention. Write reusable constructs around stable business/platform needs rather than wrapping every property mechanically. Test important synthesized invariants, such as preventing public buckets or requiring encryption. AWS gives design guidance in [CDK best practices](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html).

For SAM, learn build/package/deploy stages, function permissions, event definitions, local invocation, layers, aliases, and deployment preferences. Local emulation is useful for handler behavior; it does not fully reproduce IAM, networking, quotas, managed-service retries, or production latency. Run a small integration test in an isolated AWS environment.

## 5. Protect Terraform or OpenTofu state

State maps configuration addresses to remote resources and records attributes needed for management. It may contain secrets even if terminal output labels values sensitive. Protect state, saved plans, crash logs, and backup files as confidential artifacts. Do not commit them to Git or upload them to public issue trackers.

Use an access-controlled remote backend with encryption, versioning, auditing where required, and recovery procedures. Separate environment state by account and lifecycle. CLI workspaces help organize instances of configuration but are not, by themselves, a strong security boundary. A production deployment role should not gain access to every environment merely because they share a convenient backend.

State locking prevents conflicting writers; it does not protect against a mistaken authorized change. Current Terraform's S3 backend supports opt-in native lockfiles using `use_lockfile = true`; its DynamoDB-based locking is deprecated. OpenTofu also supports native S3 locking and currently continues to support DynamoDB locking. Use the documentation for the pinned engine/version, coordinate all clients during a migration, and never mix incompatible locking schemes against the same state. See [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) and [OpenTofu S3 backend](https://opentofu.org/docs/language/settings/backends/s3/).

This illustrative backend block is for an engine version that supports native S3 locking. Replace the bucket, key, and Region with existing approved backend resources before initialization; obtain credentials through the standard temporary-credential chain.

```hcl
terraform {
  backend "s3" {
    bucket       = "replace-with-your-private-state-bucket"
    key          = "sandbox/application/terraform.tfstate"
    region       = "replace-with-your-region"
    encrypt      = true
    use_lockfile = true
  }
}
```

Grant access to the correct state and lock objects, and any required KMS key. Create the backend through a separately owned bootstrap process so an application destroy cannot remove its own recovery records. Force-unlock only after establishing that no writer remains active and documenting the incident.

## 6. Write maintainable provider-based configuration

Learn providers, resources, data sources, variables, locals, outputs, modules, expressions, dependency inference, `for_each`, `count`, and lifecycle rules. Choose stable keys for collections. Removing an item from a list used with numeric indexes can shift resource identities; a stable business key often makes intent clearer.

Pin the CLI version and set deliberate provider version constraints. Commit provider dependency lockfiles and review updates. Module versions also need explicit control; a provider lockfile does not pin every remote module or external script. Keep documentation on supported versions and the process for upgrading them.

Understand import and moved-resource declarations before refactoring existing resources. Use a plan to confirm that renaming a module will not recreate data infrastructure. `ignore_changes` can acknowledge another legitimate owner for selected attributes, but excessive use hides drift. Targeted applies are recovery tools with incomplete dependency context, not a default release process.

Use external secret references where possible. Suppressing output is not the same as keeping a value out of state; verify how the provider handles that value. Restrict provisioning scripts and remote-exec patterns because they often make retries, failure recovery, and secret handling harder to reason about.

## 7. Establish the engineering workflow

1. **Design:** State resource ownership, environment boundaries, names/tags, data lifecycle, expected cost, and teardown behavior.
2. **Develop:** Use small modules with documented inputs/outputs and explicit dependencies. Keep environment configuration separate from reusable logic.
3. **Validate:** Format, parse/validate, lint, and scan for important security policies. Test meaningful resource invariants and failure-prone logic.
4. **Plan:** Use the actual target account and state through a narrowly scoped role. Protect the resulting plan because it can contain sensitive values.
5. **Review:** Inspect deletions, replacement, IAM changes, public exposure, network routes, data retention, and costs. Re-plan after relevant configuration or infrastructure changes.
6. **Apply:** Serialize deployments for each state/stack. In a pipeline, tie the reviewed plan and artifact to the exact source revision and environment.
7. **Verify:** Check application behavior, resource settings, policy enforcement, telemetry, and ownership tags. A successful API call is not a complete acceptance test.
8. **Maintain:** Detect drift, upgrade dependencies, rotate secrets, rehearse recovery, and retire unused environments.

Policy tools may include CloudFormation Guard, organizational controls, provider-aware scanners, or a policy engine. Define a small set of requirements and owners for exceptions. A scanner finding is evidence to evaluate; passing a scanner is not proof that an architecture is secure.

## 8. Lab: create, change, import, and recover

Use a dedicated sandbox and one tool. Build a small private S3 bucket, SQS queue, application IAM role, and CloudWatch log group with explicit retention. Tag all resources with a unique lab identifier. Keep the backend or deployment bootstrap outside this application stack.

1. Write the configuration from the official resource reference. Block public S3 access and permit the role to access only the intended prefix and queue.
2. Validate and review the first plan/change set. Deploy, then list resources and verify actual settings.
3. Make a safe update, such as log retention or a tag. Confirm the plan shows an in-place change where supported.
4. Propose a replacement-triggering change to a disposable resource and inspect it without automatically applying it. Explain the data and dependency consequences.
5. Introduce a harmless console drift, detect it, and choose whether configuration or live state should win. Record the reasoning.
6. Create a second disposable resource separately, then practice import using the tool's documented process. Confirm a clean plan afterward.
7. Simulate a failed update with an invalid sandbox setting or permission, inspect events, repair the cause, and recover without manually deleting state.
8. For a stateful engine, inspect access controls and demonstrate state-version recovery in a disposable copy. For CloudFormation, demonstrate a retained resource and explain its ownership after stack deletion.

**Verify:** A fresh deployment produces the intended environment; a second unchanged plan is empty; unauthorized access fails; drift is explained; a refactor preserves physical resources; and the teardown plan affects only lab resources.

**Cost:** Most configuration tools do not eliminate the charges of resources they create. Account for retained storage, object versions, logging, build assets, state requests, optional scanning, and any paid management platform. Estimate before deployment and use only small synthetic data.

**Cleanup:** Review a destroy/delete plan, remove lab data according to the intended retention policy, delete the application, and separately inspect retained resources. Keep shared bootstrap/state resources if other stacks use them. Delete a dedicated backend only after all managed resources are gone and required recovery evidence is preserved.

## Pitfalls and mastery checklist

Common failures include committing state, using unbounded provider versions, granting a deployment role permanent administrator access, bypassing locking, accepting destructive replacements without reading the plan, and assuming rollback restores externally modified data. Another frequent problem is building overly generic modules before understanding a second real use case.

- [ ] I can explain configuration, state, and actual resources separately.
- [ ] I can identify a replacement and its data consequences before applying.
- [ ] I can protect state and verify locking with the chosen tool/version.
- [ ] I can import or refactor resources without accidental recreation.
- [ ] I can diagnose a failed update and recover using supported procedures.
- [ ] I can explain retained resources, bootstrap ownership, and cleanup.
- [ ] My repository records dependency versions, inputs, outputs, and deployment instructions.

## Official references

- [CloudFormation user guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) and [resource reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html).
- [AWS CDK v2 guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html).
- [AWS SAM developer guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html).
- [Terraform language documentation](https://developer.hashicorp.com/terraform/language).
- [OpenTofu documentation](https://opentofu.org/docs/).
- [CloudFormation Guard](https://docs.aws.amazon.com/cfn-guard/latest/ug/what-is-guard.html).
- [Systems Manager State Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html).

**Review date:** 2026-10-18. Pin tested versions and recheck backend locking, resource support, and migration guidance when upgrading.
