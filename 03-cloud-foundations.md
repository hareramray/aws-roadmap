# 03 — Cloud foundations and AWS architecture

[Roadmap index](README.md) · Prerequisites: [technical prerequisites](01-prerequisites.md), [account and tooling](02-account-and-tooling.md)

Reviewed: **2026-10-18**. This module builds the reasoning used throughout the roadmap. Learn the boundaries and tradeoffs before memorizing product names. AWS changes continuously; check regional availability, quotas, and pricing before deploying a lab.

## Outcomes

By the end, you should be able to translate a business requirement into an initial AWS architecture, explain who operates each layer, identify failure boundaries, and estimate the major sources of cost. You should also distinguish an availability feature from a backup, a service quota from a performance limit, and an AWS account from a network boundary.

## 1. Cloud concepts that affect decisions

Cloud computing combines on-demand provisioning, network access, shared infrastructure, elastic capacity, and measured usage. These capabilities change operating practices, not just the location of servers.

- **Scalability** is the ability to handle greater demand. Vertical scaling enlarges a resource; horizontal scaling adds resources. Both have constraints.
- **Elasticity** changes capacity as demand changes. It requires suitable metrics, limits, startup time, and application behavior. Autoscaling cannot repair an inefficient database query automatically.
- **Availability** concerns whether a service can perform its intended function now. **Durability** concerns whether stored data survives. A highly durable archive may have a long retrieval time.
- **Reliability** includes correct operation, recovery, and consistent behavior over time. An HTTP endpoint that answers quickly with incorrect totals is not reliable.
- **Agility** means shortening safe delivery and learning cycles. Infrastructure as code, automated verification, and clear ownership matter as much as provisioning speed.
- **Consumption pricing** shifts some capital expenditure to operating expenditure. It does not guarantee savings: idle resources, inefficient queries, transfer, logs, and licenses still cost money.

Compare operating models through responsibility, not marketing labels:

| Model | Typical AWS example | What your team still operates |
| --- | --- | --- |
| Infrastructure service | EC2 | Guest OS, patching, application runtime, deployment, scaling policy, data protection |
| Managed platform | RDS, ECS with Fargate | Schema or application, identities, network configuration, capacity choices, recovery objectives |
| Event-driven compute | Lambda | Code, dependencies, concurrency controls, retries, permissions, observability |
| Managed application | A subscribed SaaS application | Users, data classification, configuration, integration, governance |

Serverless means less infrastructure management; it does not mean no servers, no limits, no operations, or universally lower cost.

## 2. Global infrastructure and resource scope

An AWS **Region** is a geographic deployment area containing multiple **Availability Zones**. An AZ comprises one or more discrete data centers with independent infrastructure. A Region is not one building, and an AZ is not simply a rack. [AWS global infrastructure overview](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/global-infrastructure.html)

Choose a Region using customer latency, legal or contractual data-location requirements, service and feature availability, dependencies, capacity, and cost. Record the reason in an architecture decision record. Do not choose solely because a tutorial uses it.

Learn the scope of each resource:

- IAM identities are account-level; many operational resources are Regional.
- A VPC spans a Region; a subnet belongs to one AZ.
- EC2 instances and EBS volumes are zonal. A multi-AZ application needs resources and routing across those zones.
- S3 general purpose buckets are created in a Region, although bucket names and endpoints have their own naming and access rules.
- Edge services bring delivery or execution closer to users. An edge location is not an additional application Region.
- Local Zones and Outposts address particular locality or hybrid requirements. Their supported services and failure dependencies require separate study.

AZ names such as `us-east-1a` should not be treated as universal physical identifiers across accounts. Use AZ IDs when coordinating physical placement across accounts. Never assume that enabling a Region copies resources into it.

## 3. Accounts, identity, networks, and endpoints

An AWS account is an ownership, billing, quota, and security boundary. A VPC is a network boundary within an account and Region. A subnet divides addressing and placement inside that network. None replaces an authorization policy.

For organizations, separate production, nonproduction, security tooling, and log archives where the operational benefit justifies the structure. Keep ordinary workloads out of the management account. For a personal learner, begin with the account and budget guidance in [module 02](02-account-and-tooling.md); creating an organization is not a required first lab and can affect account-plan benefits.

Understand API access: a human or workload signs a request, a service evaluates permissions and context, and an API returns a result or error. The console is another client of services. An SDK credential chain can use a different identity than expected; explicitly verify account, role, and Region before mutations.

Distinguish **control plane** operations, such as creating a load balancer, from **data plane** operations, such as serving traffic through it. Design stable workloads so that a temporary control-plane impairment does not require rebuilding essential resources to keep serving users. [AWS fault isolation boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/abstract-and-introduction.html)

## 4. Shared responsibility

AWS manages the infrastructure underlying its services. Your responsibilities depend on the service: with EC2 you manage the guest operating system; with more abstract services you still manage data, application behavior, and access configuration. An AWS compliance report does not establish that your application is compliant. [Shared responsibility model](https://aws.amazon.com/compliance/shared-responsibility-model/)

For each architecture, assign an owner to identity lifecycle, vulnerability remediation, key administration, backup restoration, logging, incident response, and cost review. A responsibility that says “managed by AWS” needs a specific service guarantee behind it.

Practice this distinction with three incidents: a stolen application credential, an unpatched package in your container, and a failed underlying host. State which layers AWS operates, which decisions remain yours, what evidence you need, and how the application recovers.

## 5. Requirements before services

Start every design with a short workload specification:

1. Users, locations, traffic shape, payload size, and expected growth.
2. Data types, sensitivity, retention, deletion, and consistency needs.
3. Latency percentiles and throughput targets; averages can conceal poor user experience.
4. Availability objective and the exact operation it measures.
5. **Recovery time objective (RTO):** acceptable time to restore service.
6. **Recovery point objective (RPO):** acceptable data loss measured in time.
7. Budget, skills, existing systems, delivery deadline, and vendor constraints.
8. Ownership, on-call expectations, operational evidence, and exit strategy.

An RPO of zero is a demanding requirement; copying yesterday's snapshot cannot meet it. A two-Region diagram also does not prove a short RTO. Recovery needs data, access, capacity, routing, dependencies, and a tested procedure.

Choose a baseline design, list assumptions, and compare alternatives. Prefer the least complicated architecture that meets verified requirements. Add multi-Region operation when its business value pays for replication, conflict handling, deployment coordination, and recovery testing.

## 6. Distributed-system behavior

Network calls can time out after the server has already performed a write. A retry can therefore duplicate work. Learn idempotency keys, deduplication, conditional updates, exponential backoff with jitter, bounded retries, and end-to-end deadlines.

Queues absorb bursts but do not create unlimited processing capacity. Track queue age alongside length. A dead-letter queue needs an owner and redrive process. Backpressure protects a downstream database; retries without limits can amplify an outage.

Separate state from replaceable compute. Use durable storage for authoritative records, explicit schemas for messages, and compatibility rules for deployment. Cache invalidation, asynchronous replication, clock skew, partial failure, and race conditions belong in design reviews.

Treat deployment, tenant, cell, account, AZ, and Region as possible fault boundaries. Ask whether supposedly independent replicas share a secret, database, NAT path, deployment pipeline, or external provider. Redundant servers with one shared dependency may still fail together.

## 7. Well-Architected review and cost model

Review six dimensions: operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability. These are complementary questions rather than a final certification stamp. [AWS Well-Architected pillars](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html)

For a small application, ask: Who receives alerts? How do permissions expire? How is a failed deploy reversed? Which measurement proves capacity? What is the monthly idle cost? Can unused data and compute be removed?

Estimate at least compute duration, storage size and operations, database capacity, load balancing, public IPv4, network gateways, cross-AZ or cross-Region traffic where applicable, internet delivery, telemetry ingestion and retention, backups, and support or licensing. Use workload units: cost per customer, per request, or per processed gigabyte. A small hourly charge becomes material when multiplied by environments and months.

Service quotas may be per account, Region, resource, or operation; some can be increased and others cannot. A quota increase is not a capacity reservation. Inspect throttling behavior and request increases before demand arrives.

## 8. Implementation workflow

Write requirements → draw request and data flows → choose fault boundaries → identify identities and permissions → estimate costs → select telemetry → describe deployment and rollback → test failure and recovery → document ownership. Revisit the design after measurements. Keep decisions in version control alongside diagrams, assumptions, and links to evidence.

## Lab A — Architecture decision exercise

**Type:** local design exercise; no deployment or AWS charge.

Design a document upload service with authenticated users, private files, asynchronous thumbnail processing, and a history of uploads. Produce a one-page design, a table of data classifications, a permission matrix, three failure scenarios, and a cost-input worksheet. Compare an EC2-based implementation with managed event-driven components; do not choose solely from a product checklist.

**Verify:** trace a file from browser to durable storage, explain what happens if processing runs twice, show how a user is prevented from downloading another user's file, and state which system is authoritative. Separate RTO/RPO assumptions from proven results. **Cleanup:** remove only disposable local drafts; there are no cloud resources.

## Lab B — Read-only resource and scope inventory

**Type:** read-only AWS exercise in the sandbox configured in module 02.

Use the console to record the active account, Region, available AZ IDs, existing VPCs, and applicable EC2 quotas. If CLI access is configured, use an explicitly selected profile:

```powershell
aws sts get-caller-identity --profile roadmap --region ap-south-1
aws ec2 describe-availability-zones --profile roadmap --region ap-south-1 --query "AvailabilityZones[].{Name:ZoneName,Id:ZoneId,State:State}"
```

Replace `roadmap` and `ap-south-1` with your configured sandbox profile and chosen Region. **Verify:** match the returned account to your intended sandbox and explain the difference between AZ name and ID. Do not interpret missing permission as an unavailable service. **Cleanup/cost:** no resources are created by these calls; do not enable additional paid discovery products for this inventory.

## Common mistakes

Treating multi-AZ as a backup, equating serverless with unlimited scale, copying a diagram without its assumptions, selecting only by hourly compute price, and expecting a budget alert to stop all charges are recurring problems. Another is calling every API failure a networking issue: distinguish authentication, authorization, throttling, validation, and service errors.

## Mastery checklist

- [ ] Explain account, Region, AZ, VPC, subnet, and edge scope with examples.
- [ ] Assign responsibility for EC2, managed database, and application layers.
- [ ] Convert business needs into latency, availability, RTO, and RPO objectives.
- [ ] Identify duplicated work and shared failure dependencies.
- [ ] Compare two architectures with cost and operational tradeoffs.
- [ ] Produce a testable design instead of an unqualified availability claim.

Next: [IAM and identity](04-iam-and-identity.md), then [networking and DNS](05-networking-and-dns.md).
