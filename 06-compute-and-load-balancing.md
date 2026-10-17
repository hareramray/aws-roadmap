# 06 — Compute, load balancing, and elastic capacity

[Roadmap index](README.md) · Prerequisites: [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md); read [storage](07-storage-and-backup.md) before persistent-volume experiments

Reviewed: **2026-10-18**. This module focuses on EC2, Elastic Load Balancing, Auto Scaling, and choosing a compute model. Container and serverless modules in the index deepen those application runtimes.

## Outcomes

Launch a secure disposable instance, explain lifecycle and storage behavior, select a compute purchasing model, operate a load-balanced service, configure meaningful health and scaling signals, and recover from a failed instance without manual rebuilding.

## 1. Choose an execution model

| Model | Good fit | Your operational burden |
| --- | --- | --- |
| EC2 | OS control, specialized software, custom networking, sustained workloads | OS, agents, image updates, capacity, application runtime |
| ECS or EKS on EC2 | Containers requiring node-level control | Cluster/workload configuration plus worker nodes |
| Fargate | Supported container workloads without managing hosts | Tasks/pods, resource sizing, identity, networking, application |
| Lambda | Event-driven, bounded function execution | Code, dependencies, concurrency, retries, timeout behavior |
| AWS Batch | Queued batch jobs with scheduling requirements | Job definition, dependencies, execution environment, cost controls |
| Lightsail or a managed application platform | Simpler hosting needs | Application and service-specific operational choices |

Choose on workload characteristics: duration, startup latency, state, custom runtime needs, burstiness, scheduling, compliance, hardware, and team experience. A background task that takes hours needs a suitable execution model; forcing it into repeated short functions can create needless complexity.

## 2. EC2 building blocks

An instance combines an AMI, instance type, network interfaces, storage, identity, security groups, and launch-time configuration. An AMI is a launch image; a running instance can drift away from it. A launch template captures reusable configuration and versions, but it does not patch already-running machines automatically.

Read instance-family naming and compare general-purpose, compute-optimized, memory-optimized, storage-optimized, accelerated, and specialized instances. Benchmark realistic application work rather than selecting solely by vCPU count. CPU architecture matters: Arm/Graviton and x86 images, native dependencies, container builds, and vendor licenses must be compatible.

Understand CPU credits on burstable families, EBS bandwidth, network performance, local disks, placement, virtualization capabilities, and capacity availability. A large CPU allocation will not fix a low-throughput storage bottleneck. For placement groups, distinguish tightly coupled low-latency workloads from spread or partition isolation goals and verify supported configurations.

## 3. Secure launch and administration

Use a trusted maintained AMI, encrypted volumes, a narrowly scoped instance profile, restrictive security groups, and IMDSv2-required metadata configuration. IMDSv2 uses a session token; SDKs should obtain credentials through supported credential providers. Do not manually copy role credentials from metadata into config files. [EC2 instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

Prefer Systems Manager Session Manager where appropriate over opening SSH or RDP to the world. Session Manager still needs an installed/running agent, instance permissions, endpoint or outbound connectivity, and permissions for the operator. Logging and encryption configuration require deliberate setup.

User data is bootstrap input, not a secret vault. Avoid placing passwords or tokens in scripts that can appear in instance configuration, logs, or images. Fetch required secrets through an authorized runtime identity. Make bootstrap actions observable and safe to retry; failures should leave useful logs and should not mark a broken application healthy.

For reproducibility, use image-building automation such as EC2 Image Builder when a maintained golden image fits the workload. Keep the application release identifiable by version or digest. Separate emergency troubleshooting from the normal deployment path so manual fixes do not become undocumented infrastructure.

## 4. Lifecycle and persistent state

Learn pending, running, stopping, stopped, shutting-down, and terminated states. Reboot is different from stop/start. Instance-store data does not survive stop/start or termination; persistent data needs a suitable durable storage system. EBS lifetime on termination depends on volume settings, so inspect `DeleteOnTermination`. [EC2 instance lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

A stopped instance can still incur charges for retained storage and other associated resources. Public addressing can change across lifecycle operations; use intentional DNS and addressing rather than hardcoding an ephemeral address. Hibernation has supported-instance and storage requirements and is not a substitute for application checkpointing or disaster recovery.

Stateless application servers should be replaceable. Put authoritative records in a database or durable store, uploaded files in an appropriate storage service, and session state somewhere that supports the chosen scaling model. A single instance's local filesystem is not a shared application database.

## 5. Purchasing and capacity

On-Demand is useful for flexible use without a long-term commitment. Savings Plans and Reserved Instances can reduce eligible usage costs under commitments, but their scope and capacity implications differ. A discount instrument is not automatically a capacity guarantee; study the particular product and reservation type.

Spot capacity can be interrupted and should suit retryable or checkpointable work. Diversify acceptable instance types and AZs, use interruption signals, and build rescheduling behavior. Do not assume a specific pool will always be available or that every job gets enough time to finish after notice.

Capacity Reservations address eligible capacity requirements; dedicated options can address isolation or licensing needs. Model compute, attached storage, network transfer, IPv4, load balancing, operating-system licenses, and idle standby capacity together. Do not buy a commitment merely to complete a learning lab.

## 6. Load balancers and traffic handling

| Type | Primary focus | Study topics |
| --- | --- | --- |
| Application Load Balancer (ALB) | HTTP/HTTPS application routing | Host/path rules, target groups, certificates, redirects, WAF integration |
| Network Load Balancer (NLB) | Supported transport-level traffic and connection handling | TCP/UDP/TLS, static addresses, source-IP behavior, health, security groups |
| Gateway Load Balancer (GWLB) | Transparent deployment of compatible network appliances | Encapsulation, appliance health, routing symmetry, endpoints |

Classic Load Balancers are previous-generation technology; learn them for existing systems and migration rather than defaulting to them for new designs. [Elastic Load Balancing documentation](https://docs.aws.amazon.com/elasticloadbalancing/)

Separate the listener, routing rule, target group, target port, and health-check path. A healthy TCP port does not prove that a database-dependent HTTP operation works. A deep health check that fails whenever one optional dependency is slow can also remove every target at once. Design readiness checks around the service's actual ability to accept work.

Know connection idle timeouts, request limits, keep-alive, deregistration delay, sticky sessions, proxy headers, TLS termination, and certificate renewal. Applications should handle termination gracefully: stop accepting work, finish bounded in-flight requests, and exit within the drain window.

ALB cross-zone behavior and NLB defaults differ; inspect both load-balancer and target-group settings instead of memorizing one rule for all types. ALB supports disabling cross-zone behavior at target-group level, while NLB distributes within each enabled AZ by default unless configured otherwise. [ALB configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [NLB configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html)

## 7. Auto Scaling as a control loop

An Auto Scaling group defines capacity boundaries and placement, launches from a template, and replaces unhealthy instances. Desired capacity is the current target; minimum and maximum constrain it. Scaling policies do not eliminate those bounds or guarantee launch capacity.

Use target tracking for a meaningful utilization signal, step scaling when different thresholds need different reactions, and scheduled scaling for known patterns. Predictive scaling can complement workloads with suitable history. Select metrics that track work per capacity unit: requests per target, queue backlog per worker, or application-specific saturation may be more useful than CPU alone.

Account for instance warmup, application initialization, health-check grace periods, scale-in protection, lifecycle hooks, mixed instances, and instance refresh. Verify that replacement instances can fetch images, secrets, and configuration during a dependency issue. Scaling from an unhealthy image reproduces the fault faster.

## 8. Implementation workflow

Create a launch template → validate a single instance → verify identity and network paths → build target groups and load balancer → add an Auto Scaling group across AZs → configure health, telemetry, and bounded scaling → test replacement → test a release and rollback → measure actual cost.

Monitor application errors and latency, load balancer response codes, healthy target count, CPU, memory and disk metrics where collected, storage latency, network saturation, scaling activities, and failed launches. Built-in EC2 metrics do not magically include every guest-OS metric; install and configure collection for what you need.

## Lab A — Secure disposable EC2 instance

**Type:** sandbox deployment. Choose a short session, review pricing, and set a teardown reminder.

1. Select a maintained AMI compatible with a small chosen instance type. Record AMI owner, architecture, Region, and image ID.
2. Use the dedicated lab VPC. Configure a path for Systems Manager: suitable interface endpoints or carefully planned outbound access. Account for endpoint/NAT costs before choosing.
3. Attach a limited instance role, encrypted root volume, no public SSH/RDP rule, and IMDSv2 required.
4. Connect through Session Manager. Inspect OS version and service status; create a small dummy file and read it back.
5. Record which data and identity survive reboot versus stop/start based on the selected storage and lifecycle settings. Perform lifecycle experiments only if the access path can reconnect safely.

**Verify:** the expected account/role is in use, administration works without open inbound management ports, and the metadata configuration requires tokens. **Cleanup:** terminate the dedicated instance; verify retained volumes, snapshots, public addresses, endpoints, NAT, and log groups individually. A terminated instance alone is not a complete teardown.

## Lab B — Replaceable web service

**Type:** optional paid deployment, or architecture exercise if avoiding load-balancer costs.

1. Bake or bootstrap a simple HTTP service exposing a static version and `/health` endpoint. Use dummy content only.
2. Create an ALB in two public AZ subnets and an application group that accepts traffic only from the ALB group. Limit client access to your test source where practical.
3. Use an Auto Scaling group with two instances across private subnets, a small bounded maximum, and a working bootstrap dependency path.
4. Register its target group, enable suitable health checks, and record successful responses.
5. Terminate one lab instance deliberately. Observe target removal, reduced capacity, replacement launch, readiness, and restored capacity. Then deploy a new launch-template version through an instance refresh and describe rollback.

**Verify:** the service recovers without hand-configuring a replacement; measure the disruption rather than claiming zero downtime. **Cleanup/cost:** remove the group and instances, load balancer, target group, launch template, temporary images/snapshots, network dependencies, and logs you no longer need. ALB, instance, storage, NAT, endpoints, IPv4, and telemetry charges are separate.

## Mastery checklist

- [ ] Choose EC2, container, function, or batch execution with reasons.
- [ ] Explain image, template, instance, role, volume, and network interface.
- [ ] Distinguish reboot, stop, hibernate, and terminate consequences.
- [ ] Select health and scaling signals that represent useful work.
- [ ] Replace an instance and roll back a release reproducibly.
- [ ] Identify every billable dependency left after compute is removed.

Further references: [EC2 guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/), [EC2 Auto Scaling guide](https://docs.aws.amazon.com/autoscaling/ec2/userguide/), [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html), [AWS Batch guide](https://docs.aws.amazon.com/batch/latest/userguide/).
