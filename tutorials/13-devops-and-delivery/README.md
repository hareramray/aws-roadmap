# DevOps and delivery tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../13-devops-and-delivery.md)

Build a verifiable artifact in [secure build and release](01-secure-build-and-release-pipeline.md), then promote and recover it in [deployment and recovery](02-deployment-and-recovery.md). Both begin locally. Connecting a repository to AWS and publishing/invoking a function are optional account changes with billable resources. No workflow is installed or triggered by reading these files.

| Roadmap topics | Worked coverage |
| --- | --- |
| Git, protected branches, review, CI/CD distinctions, environment boundaries | Tutorial 1 creates tests and a deterministic artifact and separates untrusted pull-request checks from an explicitly requested protected publish job. |
| CodePipeline, CodeBuild, CodeDeploy, CodeConnections, CodeCommit, GitHub Actions, external repositories | Tutorial 1 gives a runnable local build, GitHub workflow template, CodeBuild buildspec, and AWS-native stage mapping; availability caveats are explicit. |
| CodeArtifact/ECR/S3 artifact stores, package locks, digests, SBOM/provenance/signatures | Tutorial 1 hashes the exact ZIP and manifest and explains how the same identity follows a container/package through promotion. |
| OIDC federation, short-lived sessions, trust subject/audience, scoped IAM, fork safety, pass-role, secrets | Tutorial 1 includes a constrained trust-policy template and a negative trust test, with current GitHub subject-format handling. |
| Rolling, blue/green, canary, immutable releases, feature flags, aliases, alarms, rollback | Tutorial 2 publishes immutable Lambda versions, routes a small canary, inspects executed versions, and restores the prior alias. ECS/Kubernetes adaptations are linked. |
| Expand/contract migrations, backward compatibility, data rollback limits | Tutorial 2 executes a local SQLite migration and demonstrates why deleting an old column prevents safe application rollback. |
| Unit/contract/integration/smoke/performance/security tests; production signals; approval evidence | Tutorial 1 supplies unit and artifact checks; tutorial 2 records business invariants, error counts, latency, and version identity before promotion. |
| Pipeline queues/concurrency, drift, credential failures, logs, retention, recovery permissions | Both tutorials bound operations, preserve failure evidence, and specify who can stop/restore a release. |
| Cost and teardown | Both distinguish CI minutes/artifact storage from AWS deployment costs and remove only dedicated pipeline identities/resources. |

Continue with [IaC](../12-infrastructure-as-code/README.md) for infrastructure plans and [container tutorials](../11-containers-and-kubernetes/README.md) for ECS/EKS release behavior. A pipeline success badge is evidence that its defined checks passed; it is not proof that customer-facing behavior is healthy.
