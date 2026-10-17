# 11. Containers, Amazon ECS, and Amazon EKS

[Roadmap index](README.md)

**Prerequisites:** [Linux and development foundations](01-prerequisites.md), [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md), [compute](06-compute-and-load-balancing.md), [storage](07-storage-and-backup.md), and [security](09-security-and-compliance.md). Pair this module with [IaC](12-infrastructure-as-code.md) and [CI/CD](13-devops-and-delivery.md).

**Outcome:** Package a secure application, operate it on ECS, understand Kubernetes primitives, and make an evidence-based decision about EKS. Demonstrate deployment, health checking, workload identity, scaling, rollback, and cleanup.

## 1. Understand the container before the orchestrator

A container image packages an application and its userspace dependencies. Containers still depend on a host kernel, runtime, network, storage, and architecture. An image built for one CPU architecture may not run on another unless a compatible image or emulation is available. An image is not a miniature virtual machine with its own independent kernel.

Practice a small HTTP application locally. Write a multi-stage Dockerfile, pin a maintained base image, run as a nonroot user, copy only required files, and use a `.dockerignore`. Keep credentials out of layers, build arguments, and source control. Deleting a secret in a later image layer does not remove it from the earlier layer. Use build-time secret mechanisms if a dependency requires credentials.

Learn image layers, tags versus immutable digests, registries, manifests, image signing/provenance, software bills of materials, and vulnerability scanning. Keep the runtime image small enough to inspect and update without removing tools required by the application. Rebuild routinely when dependencies or the base image change; a scan at initial release is insufficient.

Implement graceful shutdown. Stop accepting new work, finish or release in-flight work, and exit within the configured grace period. Write logs to standard output and standard error. Store durable data outside the container's writable layer. Define health behavior carefully: a dependency outage should not automatically trigger a destructive restart loop for every replica.

## 2. Learn the AWS container choices

| Choice | What you manage | Good reason to choose it |
| --- | --- | --- |
| ECS with Fargate | Task definitions, services, networking, IAM, scaling, and application operations | AWS-native containers without managing worker hosts |
| ECS on EC2 | ECS configuration plus host capacity, patching, and placement | Host-level control or specialized hardware/placement requirements |
| EKS with managed node groups | Kubernetes objects and ecosystem plus worker configuration/lifecycle | A real need for Kubernetes APIs and compatible tooling |
| EKS Auto Mode | Kubernetes workloads and policies, with more infrastructure management delegated | Kubernetes requirements with a more managed compute/network/storage path |
| EKS on Fargate | Supported Kubernetes workloads and Fargate profiles | Suitable isolated workloads after checking platform constraints |
| AWS Batch | Job definitions, queues, dependencies, and compute policy | Finite batch work requiring scheduling rather than always-running services |
| Self-managed Kubernetes on EC2 | Control plane and workers as well as workloads | A justified requirement that managed Kubernetes cannot meet |

Choose Kubernetes for requirements, ecosystem, or organizational capability, not because a container automatically requires it. Count the operational cost of cluster upgrades, admission control, networking, and platform ownership. For simple AWS applications, ECS can satisfy many of the same business outcomes with fewer moving parts.

As of this roadmap's review date, **App Runner is closed to new customers**, and AWS points users toward ECS Express Mode. Express Mode can provision an ECS application stack, but learn and inspect its underlying resources and costs. **App Mesh reached its announced end of support on September 30, 2026**, so it is not a new-build learning dependency. Learn ECS Service Connect and assess current service-connectivity options instead. See [App Runner availability](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html) and the [App Mesh notice](https://docs.aws.amazon.com/app-mesh/latest/userguide/).

## 3. Build a complete ECS mental model

Learn clusters, task definitions, task revisions, tasks, services, capacity providers, launch configuration, deployment settings, placement, load balancer integration, service discovery, and Service Connect. A task is a running unit; a service maintains the desired population of tasks. Scheduled or one-off tasks do not need an always-running service.

Separate identities. The **task execution role** permits supporting actions such as image pulls and configured log or secret retrieval. The **task role** supplies the application's AWS permissions. EC2 container instances have their own host role. Avoid assigning application permissions to the host for convenience. AWS documents these responsibilities in [ECS IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security-iam-roles.html).

With `awsvpc` networking, plan task IP addresses and security groups. A running task needs a working path to its registry, logs, secrets, and dependencies. Private subnets may need NAT or service endpoints. An ALB health check requires the correct path, port, security-group path, startup allowance, and response code. Test these dependencies before debugging application code.

Set CPU and memory according to measured resource use and compatible task configurations. Understand application autoscaling separately from EC2 capacity scaling. For queue workers, backlog per worker and message age may be more useful than CPU alone. Use interruption-aware processing for Spot capacity, maintain essential baseline capacity where necessary, and make graceful shutdown real.

Learn rolling deployments, available blue/green strategies, health-based rollback, and deployment alarms. A new task definition is an immutable release description; tag it with source and image identity. Verify supported deployment features for the chosen controller rather than combining settings from incompatible tutorials.

## 4. Learn Kubernetes locally before paying for a cluster

Use a local Kubernetes environment for the first exercises. Study the declarative reconciliation model: controllers continually attempt to make actual state match desired state.

- **Pod:** One or more tightly coupled containers sharing a lifecycle and network namespace.
- **Deployment and ReplicaSet:** Manage replicas and application rollouts.
- **Service and DNS:** Give clients a stable discovery endpoint despite changing pods.
- **Ingress or Gateway API implementation:** Express routing to services; a suitable controller must implement it.
- **ConfigMap and Secret:** Supply configuration. Base64 encoding in a Secret is not encryption or an authorization strategy.
- **ServiceAccount and RBAC:** Establish Kubernetes workload identity and permissions inside the cluster.
- **Namespace, ResourceQuota, and LimitRange:** Organize workloads and limit resource consumption; namespaces alone do not provide strong tenant isolation.
- **Job and CronJob:** Run finite and scheduled work with explicit retry/concurrency rules.
- **StatefulSet, PersistentVolume, PVC, and StorageClass:** Model persistent applications and storage provisioning.
- **DaemonSet:** Run node-level workloads on eligible nodes.

Understand labels, selectors, annotations, scheduling, taints/tolerations, affinity, topology spread, requests, limits, probes, and disruption budgets. Requests influence scheduling; limits constrain runtime consumption. CPU throttling and memory termination have different symptoms. Readiness controls whether a pod should receive traffic; liveness detects a condition where restart is useful; startup probes protect slow initialization.

## 5. Add EKS-specific networking and security

EKS manages the Kubernetes control plane, but cluster configuration and workload operation still need owners. Learn endpoint access, cluster access entries, AWS IAM authentication, Kubernetes authorization, audit/control-plane logs, add-ons, and node lifecycle. Keep emergency access documented and tested without distributing a permanent administrator credential.

Amazon VPC CNI normally assigns pod addresses from VPC address space. Node density therefore depends on subnet address availability, instance networking capacity, and CNI configuration. Plan for deployment surges, system pods, additional nodes, and failure recovery. Study prefix delegation and IPv6 where appropriate rather than expanding subnets blindly. Diagnose pending pods by separating scheduler capacity, IP allocation, image-pull, and IAM problems. AWS explains these mechanics in [EKS networking](https://docs.aws.amazon.com/eks/latest/best-practices/networking.html).

Apply network policies through a supported, configured enforcement mechanism. A YAML NetworkPolicy alone does not guarantee enforcement. Understand security groups, route tables, network ACLs, service routing, and Kubernetes policies as distinct layers. Restrict public control-plane exposure where the operational model permits, and provide a reliable administration path to private endpoints.

Give workloads AWS permissions through EKS Pod Identity or IAM roles for service accounts, according to compute compatibility and requirements. Confirm SDK support and use its default credential chain. Restrict pod access to node credentials; workloads should not inherit broad node permissions. Pod Identity associates an IAM role with a service account and needs its supported runtime components; it does not make containers a security boundary. See [Pod Identity documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html).

Use Pod Security Admission and, when needed, policy engines to constrain privileged containers, host mounts, capabilities, and unsafe execution. Patch nodes and controllers. Restrict who can create workloads using privileged service accounts. Treat permission to create pods as potentially powerful because a pod may access mounted secrets and service-account permissions.

## 6. Operate storage, scaling, and upgrades

Use supported CSI drivers for persistent volumes. Understand an EBS volume's Availability Zone relationship, attachment modes, filesystem recovery, and scheduling implications. Evaluate EFS for suitable shared-file requirements. A PVC is not a backup; test application-consistent recovery. Prefer managed databases when Kubernetes adds no benefit to the data service.

Distinguish Horizontal Pod Autoscaler, vertical resource tuning, and node autoscaling. A pod autoscaler can request replicas while insufficient node capacity leaves them pending. Karpenter or other supported capacity management can supply nodes, but disruption rules, instance availability, Spot handling, and workload constraints still govern recovery.

Plan upgrades continuously. Inventory deprecated APIs, controllers, webhooks, admission policies, CNI, DNS, proxy, CSI drivers, metrics components, node images, and application compatibility. Read supported version skew and the current EKS upgrade guidance. Rehearse in a nonproduction cluster, upgrade in a documented order, drain nodes gradually, and verify workloads afterward. Managed add-ons are not universally updated automatically; ownership differs between standard EKS and Auto Mode. See [EKS upgrade practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html) and [VPC CNI lifecycle](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html).

## 7. Lab A: ECS service from a clean image

1. Build a tiny API locally with `/health` and `/version`, graceful shutdown, and structured logging. Run it without root and with a read-only root filesystem where compatible.
2. Push the image to a private ECR repository. Record its digest, scan results, and dependency inventory. Configure lifecycle rules only after understanding what tags and images a rollback needs.
3. Provision a Fargate service, separate task and execution roles, log group retention, and an ALB using IaC. Allow the application port only from the ALB security group.
4. Provide required network access to image, logging, and secret services. Use a synthetic secret to demonstrate controlled retrieval.
5. Deploy a second version by digest and monitor rollout health. Create a failing health-check release in the sandbox and demonstrate recovery to the healthy version.
6. Run a short, bounded load test; adjust resource allocation and scaling based on measured behavior.

**Verify:** Requests return the expected release ID; healthy targets match running tasks; an unauthorized task role action fails; logs correlate requests; termination stops traffic before exit; a replacement task preserves service availability.

**Cost:** Include Fargate or EC2 capacity, ALB, logs, ECR storage/scanning, NAT/endpoints, and data transfer. An idle load balancer or NAT gateway can cost more than a tiny learning application. Use current regional pricing.

**Cleanup:** Delete the service and its task instances, then load balancer/listeners/target groups, dedicated cluster, application network resources, log groups, and unused ECR images/repository. Check IaC-retained resources and detach only lab-owned IAM policies.

## 8. Lab B: EKS operations exercise

Start locally with a Deployment, Service, ConfigMap, resource requests/limits, probes, and a disruption budget. Demonstrate rollout, rollback, pod deletion/replacement, a failed image pull, and a namespace-scoped RBAC denial.

For the optional paid extension, provision a short-lived EKS cluster with a supported version and a documented compute model. Enable required logs, configure workload identity, deploy the application, and access one narrowly scoped S3 prefix. Install only the controllers and add-ons you need. Reproduce an IP or resource-capacity problem in a controlled way and explain the observed events. Rehearse an upgrade in a disposable environment after checking the compatibility matrix.

**Verify:** Use `kubectl get`, `describe`, `logs`, rollout status, and events to explain state. Confirm AWS access succeeds for the intended service account and fails for another. Confirm policy enforcement and actual disruption behavior rather than merely showing configuration files.

**Cost and cleanup:** EKS has cluster-related charges in addition to compute and attached resources; extended support and optional management features may add charges. Delete Kubernetes-created load balancers and persistent volumes deliberately before deleting the cluster, then inspect node groups, EC2 instances, EBS volumes/snapshots, ENIs, logs, and dedicated networking. A deleted cluster does not prove every associated resource is gone.

## Common pitfalls and mastery

Avoid deploying `latest`, treating a namespace as a complete security boundary, confusing execution roles with application roles, putting databases on ephemeral container storage, and leaving a cluster upgrade until its support deadline. Do not solve every probe failure by increasing timeouts; determine whether startup, connectivity, dependency health, or the probe itself is wrong.

- [ ] I can explain ECS versus EKS in terms of operational responsibility and requirements.
- [ ] I can build and deploy an image by immutable digest without embedded credentials.
- [ ] I can explain task roles, node roles, Kubernetes RBAC, and workload IAM separately.
- [ ] I can diagnose a pending pod and an unhealthy ECS task from evidence.
- [ ] I have demonstrated graceful termination, rollback, and narrow AWS access.
- [ ] I can plan subnet capacity, storage placement, and autoscaling together.
- [ ] I can describe a tested upgrade and cleanup procedure.

## Official references

- [ECS developer guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html).
- [ECR user guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html).
- [EKS best practices](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html).
- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html).
- [Kubernetes concepts](https://kubernetes.io/docs/concepts/) and [security checklist](https://kubernetes.io/docs/concepts/security/security-checklist/).
- [Docker build best practices](https://docs.docker.com/build/building/best-practices/).

**Review date:** 2026-10-18. Recheck Kubernetes support windows, add-on compatibility, platform limitations, and service lifecycle notices before following older workshops.
