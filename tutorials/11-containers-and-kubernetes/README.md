# Containers and Kubernetes tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../11-containers-and-kubernetes.md)

Build the image once in [ECS container service](01-ecs-container-service.md), then reuse it in [Kubernetes and EKS operations](02-kubernetes-and-eks-operations.md). The first exercise can run entirely in Docker; the second requires an existing local Kubernetes environment or an explicitly funded EKS sandbox. A managed cluster is not required to understand reconciliation.

| Roadmap coverage | Tutorial and evidence |
| --- | --- |
| Image layers, base digest, architecture, Dockerfile, nonroot operation, graceful shutdown, registry, SBOM/signing/scanning | Tutorial 1 builds a standard-library HTTP application, checks its user and health, pushes an immutable image, and records a release manifest. |
| ECS clusters/task definitions/tasks/services; Fargate and EC2 capacity; IAM execution/task/host roles | Tutorial 1 deploys a complete Fargate service stack and separates its identities. Capacity comparisons explain which additional host duties EC2 introduces. |
| `awsvpc`, task IPs/security groups, registry/log dependencies, ALB, Service Connect, discovery | Tutorial 1 uses a constrained direct test endpoint and includes a precise ALB/Service Connect production extension and networking failure diagnosis. |
| CPU/memory, health, deployment, rollback, autoscaling, Spot, Batch | Tutorial 1 measures a process, changes desired count, injects an invalid revision, and maps finite work to Batch and interruption-safe workers. |
| Kubernetes Pods, Deployments, Services, configuration, secrets, probes, resource requests/limits, RBAC | Tutorial 2 deploys an explicit manifest and observes rollouts, readiness, authorization, and resource behavior. |
| StorageClasses/PVCs, EBS/EFS CSI, zone topology, snapshots, stateful applications | Tutorial 2 includes a PVC failure/placement exercise and separates data backup from controller recreation. |
| EKS access entries, endpoint access, OIDC/IRSA/Pod Identity, IMDS boundaries, VPC CNI/IP allocation, network policy | Tutorial 2 verifies these independently and supplies the Pod Identity association command. |
| Managed node groups, Auto Mode, EKS Fargate, node scaling, HPA, Karpenter, managed add-ons | Tutorial 2 compares operational ownership and requires capacity/metrics evidence before scaling. |
| PDB, disruption, draining, Kubernetes/API/add-on upgrades, rollback/rebuild | Tutorial 2 breaks readiness, restores an image, and writes/executes a staged upgrade checklist. |
| App Runner availability, App Mesh retirement, ECS Express Mode | Tutorial 1 explains why the exercise uses supported ECS primitives; no retired service is a prerequisite. |
| Cost, networking and storage leftovers, cleanup | Both tutorials enumerate resources that outlive workloads and require final inventory checks. |

Prerequisites: [networking](../../05-networking-and-dns.md), [IAM](../../04-iam-and-identity.md), and basic PowerShell/Python. Do not apply a Kubernetes manifest until you have verified the current context points at the intended sandbox. Do not overwrite a team's cluster configuration to reproduce a learning exercise.
