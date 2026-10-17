# Compute and load-balancing tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../06-compute-and-load-balancing.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — Secure instance and lifecycle](01-secure-instance-and-lifecycle.md) | Managed access, reproducible bootstrap and storage-lifecycle evidence |
| [02 — Load-balanced replaceable service](02-load-balanced-replaceable-service.md) | ALB/Auto Scaling service with replacement and rollout tests |
| [03 — Select capacity and execution models](03-capacity-and-execution-models.md) | Compute decision, scaling simulation and complete cost worksheet |

## Coverage mapping

| Original headings and subtopics | Tutorial |
| --- | --- |
| Execution models: EC2, ECS/EKS on EC2, Fargate, Lambda, Batch, Lightsail/managed hosting | 03 |
| Building blocks: AMI, instance type/family, Graviton/x86, credits, EBS/network limits, placement | 01, 03 |
| Secure launch: roles, IMDSv2, SSM, encrypted disks, user data, image building | 01 |
| Lifecycle: reboot/stop/hibernate/terminate, instance store/EBS, addressing and statelessness | 01 |
| Purchasing: On-Demand, Savings Plans/RI, Spot, Capacity Reservations, dedicated/licensing | 03 |
| Load balancing: ALB/NLB/GWLB, Classic recognition, listeners/rules/health/TLS/draining/stickiness/cross-zone | 02 |
| Auto Scaling: desired/min/max, target/step/scheduled/predictive, warmup, hooks/protection, mixed instances/refresh | 02, 03 |
| Workflow/telemetry, secure instance lab, replaceable web lab | 01, 02 |
| Coverage-map additions: Elastic Beanstalk, Image Builder, accelerated compute/EFA | 01, 03 |
| Mastery and common mistakes | Verification and negative checks in all |
