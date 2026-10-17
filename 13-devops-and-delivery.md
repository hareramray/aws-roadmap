# 13. DevOps, CI/CD, and software delivery

[Roadmap index](README.md)

**Prerequisites:** [development basics](01-prerequisites.md), [IAM](04-iam-and-identity.md), [security](09-security-and-compliance.md), [serverless](10-serverless-and-integration.md) or [containers](11-containers-and-kubernetes.md), and [IaC](12-infrastructure-as-code.md).

**Outcome:** Build a pipeline that produces traceable artifacts, deploys with temporary credentials, validates health, and recovers safely. Explain how source changes become running software and how an incident can be traced back to a release.

## 1. Start with delivery principles

DevOps is a way of sharing responsibility for software delivery and operation. A pipeline is one mechanism, not the whole practice. Teams also need clear service ownership, documented interfaces, feedback from production, manageable on-call work, and a process for improving recurring failures.

Continuous integration means integrating changes frequently and validating them automatically. Continuous delivery keeps changes releasable through an automated path; continuous deployment extends that path to production without a manual release decision for every change. Choose the release policy according to risk and evidence, not a slogan.

Learn small changes, trunk-based development or a justified branching model, pull-request review, protected branches, versioned dependencies, reproducible builds, release notes, and feature flags. A feature flag separates deploying code from exposing behavior, but it requires ownership, expiry, access control, and testing of relevant combinations.

Measure outcomes such as change lead time, deployment frequency, failed changes, and recovery time alongside customer reliability. Do not optimize a delivery metric by shipping risky changes or redefining incidents. Find the slowest or most error-prone step and improve it with evidence.

## 2. Understand the service roles

| Capability | AWS or compatible option | What to learn |
| --- | --- | --- |
| Source hosting | CodeCommit, GitHub, GitLab, other Git providers | Repository permissions, branch rules, review, signing where required, recovery |
| Pipeline orchestration | CodePipeline or a repository-hosted workflow engine | Triggers, dependencies, stages, artifacts, execution concurrency, gates |
| Build/test execution | CodeBuild or appropriately secured runners | Build environment, cache, network access, service role, logs, reports |
| Package repository | CodeArtifact | Domains/repositories, upstream sources, package permissions, version retention |
| Container registry | ECR | Digests, scanning, lifecycle, replication, release retention |
| Deployment control | CodeDeploy, CloudFormation/CDK/SAM, ECS/EKS deployment tooling | Health validation, traffic shift, permissions, rollback |
| Source connection | AWS CodeConnections where supported | Installation authorization, repository scope, connection permissions |
| Host/image lifecycle | EC2 Image Builder and Systems Manager | Tested images, patching, configuration, inventory, controlled automation |

CodeBuild executes build steps; CodePipeline coordinates a release workflow. CodeDeploy supports particular compute/deployment models, and its configuration is not interchangeable with every ECS or Kubernetes deployment controller. Choose the deployment mechanism for the target rather than forcing one service into every architecture. See [CodeBuild overview](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html) and [CodePipeline overview](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html).

**Current service status:** CodeCommit reopened to new customers on November 24, 2025; older advice saying it cannot be adopted is outdated. CodeCatalyst closed to new customers on November 7, 2025, and existing customers cannot create new spaces. This roadmap uses established pipeline building blocks rather than requiring CodeCatalyst. See [CodeCommit announcement](https://aws.amazon.com/blogs/devops/aws-codecommit-returns-to-general-availability/) and [CodeCatalyst migration guidance](https://docs.aws.amazon.com/codecatalyst/latest/userguide/migration.html).

## 3. Build once, promote the same artifact

A useful pipeline has explicit stages:

1. **Source:** Identify the exact commit, trusted trigger, and dependency locks.
2. **Fast validation:** Format/lint, type checking where useful, unit tests, and template validation.
3. **Security and dependency checks:** Secret scanning, dependency analysis, infrastructure policies, and image scanning where relevant.
4. **Build:** Produce a package, image, or template using pinned dependencies and a controlled build image.
5. **Publish:** Store an immutable artifact with digest, source commit, build identity, dependency information, and provenance.
6. **Deploy to test:** Provision or update an isolated environment with the same release candidate.
7. **Integration and smoke tests:** Verify critical paths, authorization, migrations, telemetry, and failure handling.
8. **Promote:** Move the tested artifact through environments using environment-specific configuration and permissions.
9. **Observe and decide:** Watch release health, complete rollout or revert/roll forward, then record the outcome.

Avoid rebuilding separately for production because the resulting artifact can differ from the tested one. Keep configuration outside the immutable application artifact when practical. Treat dependency downloads, build tools, and base images as supply-chain inputs. Record versions; verify artifacts from trusted sources; pin third-party workflow actions to reviewed immutable revisions when possible.

Use caches to reduce work, not as the authoritative artifact store. Key caches by relevant dependencies and platform. A poisoned or stale cache should not silently change release contents. Separate untrusted pull-request caches and artifacts from privileged production release inputs.

## 4. Make pipeline identity and trust explicit

Use workload roles or federation instead of storing long-lived AWS access keys in a repository's secret store. For GitHub Actions, an OIDC token can be exchanged for temporary role credentials. Restrict the role trust policy to the intended audience and repository/workflow context, then apply a narrow permissions policy to the role.

Verify the actual subject-claim format. GitHub's current documentation describes immutable owner/repository IDs for repositories created after July 15, 2026 or opted into that format. Environment-based subjects also differ from branch-based subjects. Copying an old name-only policy can fail or describe the wrong trust boundary. Use [GitHub's AWS OIDC guide](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) to construct the precise conditions for your repository.

Separate roles for build, artifact publication, nonproduction deployment, and production deployment. A build role generally should not administer IAM or modify production databases. Apply environment protections and source rules to privileged jobs. Prevent untrusted fork code from running with deployment credentials, production secrets, or access to sensitive self-hosted runners.

Treat a workflow definition as executable privileged code. Changing the build script can be equivalent to obtaining the build role's permissions. Review pipeline changes, restrict who can bypass branch rules, and consider who can modify reusable workflows, runner images, deployment templates, and package upstreams.

In a cross-account pipeline, document every trust edge: pipeline service role, deployment role, artifact bucket policy, KMS permissions, registry access, and target service permissions. Encryption does not grant authorization; both object access and key access may be necessary. Use CloudTrail to verify which principal actually performed a deployment.

## 5. Choose a deployment strategy

| Strategy | Useful properties | Main tradeoff |
| --- | --- | --- |
| All at once | Simple for disposable or very low-risk workloads | Largest immediate exposure and possible interruption |
| Rolling | Gradually replaces instances, tasks, or pods | Old and new versions coexist; capacity and compatibility matter |
| Blue/green | Keeps old and new environments for a controlled switch | Temporary duplicate capacity and synchronization complexity |
| Canary | Exposes a small population or traffic share first | Requires representative routing and meaningful health signals |
| Linear traffic shift | Increases exposure in fixed increments | Needs thresholds, observation windows, and safe stop behavior |
| Feature-flag rollout | Controls a behavior separately from deployment | Adds configuration, audit, and cleanup responsibility |

Design health signals before choosing rollout percentages. An HTTP 200 on `/health` does not prove users can place orders. Monitor errors, latency, saturation, queue age, and business outcomes. Compare new and old versions when possible. Low-traffic services may require synthetic transactions or a longer observation window.

Rollback is not universally safe. A binary rollback does not undo a destructive schema migration or an external side effect. Use backward-compatible expand-and-contract database changes: add the new structure, deploy compatible readers/writers, backfill and verify, migrate access, and remove the old structure in a later release. Maintain a roll-forward option when reverting code would make the data problem worse.

## 6. Test the release path intelligently

Unit tests validate isolated logic; contract tests validate interfaces; integration tests validate service interactions; smoke tests quickly validate essential deployed behavior; load tests explore capacity and performance; resilience exercises examine failure and recovery. Select the smallest set that can detect the meaningful risks of a change.

Use synthetic data and isolated dependencies for ordinary pipeline tests. Make test cleanup reliable and observable. A failed cleanup should produce a report listing resources, not disappear behind a successful build result. Put maximum duration and concurrency controls on temporary environments so a broken trigger cannot create an unlimited fleet.

Manage flaky tests as engineering defects. Repeatedly rerunning until a test passes masks evidence. Separate infrastructure availability failures from product regressions, but keep both visible. Capture structured reports and enough sanitized logs to debug without exposing secrets.

For infrastructure, validate syntax, inspect plans, check policy invariants, deploy a representative sandbox, and test important permissions. Avoid creating a giant production clone for every documentation change. Tie validation depth to the affected behavior and deployment risk.

## 7. Operate the delivery platform

Create a release record containing artifact digest, commit, configuration version, environment, migration status, operator or workload identity, deployment time, validation result, and rollback/roll-forward reference. Add release markers to dashboards and include version metadata in application logs.

Set concurrency rules per environment so two releases do not race database migrations or state updates. Decide whether superseded builds should be canceled and whether a running deployment is safe to interrupt. Handle webhook duplication and repeated release requests idempotently.

Monitor the pipeline itself: queue time, failed builds, expired connections, dependency outages, runner capacity, artifact-storage growth, and unexpected credential use. Patch self-hosted runners and avoid persistent shared runners for untrusted code. Prefer short-lived execution environments when feasible.

Build runbooks for failed rollout, revoked source connection, compromised dependency, broken production role, artifact rollback, and pipeline outage. Protect emergency access, log its use, and reconcile any emergency manual change back into source-controlled configuration.

## 8. Lab: delivery of a small API

Use the application from the serverless or container module. Start with a single sandbox environment; add a second isolated environment to demonstrate promotion without using a real production account.

1. Create a Git repository with application code, infrastructure code, dependency locks, a test suite, and a deployment guide.
2. Configure a source-triggered pipeline using CodePipeline/CodeBuild or a repository workflow. Use temporary credentials and narrow roles.
3. Run linting, meaningful tests, dependency checks, and infrastructure validation. Fail the run on an intentionally broken test.
4. Build an immutable artifact, record its digest and commit, and publish it to the appropriate private artifact store.
5. Generate a deployment plan and apply it to the sandbox through the pipeline's deployment role. Do not grant the build stage broad deployment access.
6. Run a smoke test that creates and retrieves a synthetic record. Validate a forbidden access attempt as well as the happy path.
7. Deploy a release with a deliberately failing health condition. Demonstrate that health checks stop promotion or trigger the configured recovery.
8. Promote the original tested artifact into the second environment. Show that its digest matches the first environment's candidate.
9. Add a compatible schema or configuration change and document why both old and new versions can coexist.

**Verify:** A source commit maps to one traceable artifact and deployment; an unauthorized branch/context cannot assume the deployment role; logs contain no credential values; failed tests stop deployment; health failure is visible; and a controlled recovery restores the expected version and behavior.

**Evidence:** Save pipeline definitions, sanitized role policies, a release manifest, test results, failure/recovery screenshots, and an architectural note explaining trust boundaries. Record the recovery duration and what required manual action.

**Cost:** Account for build execution, pipeline pricing model, runner hosts, artifact/package storage, logs, image scanning, test environments, and duplicate rollout capacity. Put retention policies on artifacts only after establishing the required rollback window.

**Cleanup:** Disable triggers, stop pending runs, delete temporary application environments, and remove dedicated pipeline/build/deployment resources. Remove dedicated source connections or trust roles only if unused elsewhere. Clean old artifacts, images, packages, log groups, and test data; check whether repository hosting has separate billing.

## Pitfalls and mastery checklist

Avoid using administrator credentials to fix a failing build, allowing unreviewed code to run on a privileged runner, rebuilding during production promotion, relying only on process health, and equating rollback with data recovery. Avoid blanket wildcard trust conditions merely to make OIDC work.

- [ ] I can trace a running release to its commit, artifact, identity, and configuration.
- [ ] My pipeline uses temporary credentials with explicit trust conditions.
- [ ] Untrusted source changes cannot access production roles or secrets.
- [ ] I can explain build, orchestration, deployment, and artifact storage separately.
- [ ] I have demonstrated a failed validation and a failed rollout recovery.
- [ ] I can design a backward-compatible database migration.
- [ ] I can reproduce the pipeline and remove its lab resources from code.

## Official references

- [CodePipeline user guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html).
- [CodeBuild user guide](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html).
- [CodeDeploy user guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html).
- [CodeArtifact user guide](https://docs.aws.amazon.com/codeartifact/latest/ug/welcome.html).
- [AWS CodeConnections](https://docs.aws.amazon.com/dtconsole/latest/userguide/welcome-connections.html).
- [AWS Well-Architected DevOps Guidance](https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/welcome.html).
- [GitHub Actions security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).

**Review date:** 2026-10-18. Recheck source-provider authentication, workflow subject claims, supported deployment strategies, and current service availability before implementation.
