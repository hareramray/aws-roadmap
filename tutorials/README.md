# AWS tutorials: the complete roadmap companion

[Start here](START-HERE.md) · [Topic coverage register](COVERAGE.md) · [Original AWS roadmap](../README.md)

This folder turns the roadmap into worked tutorials: concepts followed by specific steps, code/configuration or design calculations, expected results, failure exercises, troubleshooting, cost considerations, and cleanup. Every original roadmap chapter has a tutorial route. Subject indexes map its subtopics to the relevant lessons.

## Subject folders

| Order | Subject | What you practice |
|---|---|---|
| 01 | [Prerequisites](01-prerequisites/README.md) | Shell/Linux, networking, Python, Git, SQL, containers, distributed-system behavior |
| 02 | [Account and tooling](02-account-and-tooling/README.md) | Secure setup, credentials, budgets, CLI/SDK, private S3 and SQS |
| 03 | [Cloud foundations](03-cloud-foundations/README.md) | Resource scope, shared responsibility, requirements, failure domains |
| 04 | [IAM and identity](04-iam-and-identity/README.md) | Policies, federation, trust, roles, conditions, workload/customer identities |
| 05 | [Networking and DNS](05-networking-and-dns/README.md) | VPC routes, IPv4/IPv6, endpoints, DNS, edge, hybrid connectivity |
| 06 | [Compute and load balancing](06-compute-and-load-balancing/README.md) | EC2, secure management, health checks, scaling, capacity choices |
| 07 | [Storage and backup](07-storage-and-backup/README.md) | S3, versioning, block/file stores, movement, protected restore |
| 08 | [Databases and caching](08-databases-and-caching/README.md) | Relational models, DynamoDB, consistency, caches, specialist databases |
| 09 | [Security and compliance](09-security-and-compliance/README.md) | Keys/secrets, network protection, detection, response, evidence |
| 10 | [Serverless and integration](10-serverless-and-integration/README.md) | Idempotent events, APIs, queues, workflows, failure handling |
| 11 | [Containers and Kubernetes](11-containers-and-kubernetes/README.md) | ECS deployment, Kubernetes/EKS identity, networking, storage, upgrades |
| 12 | [Infrastructure as code](12-infrastructure-as-code/README.md) | CloudFormation/CDK/SAM, Terraform/OpenTofu, state and change review |
| 13 | [DevOps and delivery](13-devops-and-delivery/README.md) | Secure builds, artifact promotion, deployment strategies, rollback |
| 14 | [Data engineering and analytics](14-data-engineering-and-analytics/README.md) | Governed lakes, batch/stream processing, warehouses, replay and quality |
| 15 | [AI, ML, and generative AI](15-ai-ml/README.md) | 18 lessons spanning SageMaker, Bedrock, RAG, agents, security, evaluation, specialized APIs |
| 16 | [Observability and operations](16-observability-and-operations/README.md) | Metrics/logs/traces, OTel, SLOs, configuration, operational response |
| 17 | [Reliability and disaster recovery](17-reliability-and-disaster-recovery/README.md) | Restore evidence, recovery objectives, regional failure, failback |
| 18 | [Architecture and system design](18-architecture-and-system-design/README.md) | Capacity calculations, tradeoffs, tenancy, transactional/event design |
| 19 | [Organizations and governance](19-organizations-and-governance/README.md) | Accounts/OUs, controls, landing zones, central evidence and exceptions |
| 20 | [Cost, FinOps, and performance](20-cost-finops-and-performance/README.md) | Billing/unit costs, budgets, commitments, measured optimization |
| 21 | [Migration, hybrid, and edge](21-migration-hybrid-and-edge/README.md) | Assessment, cutover/rollback, hybrid networking, edge/VMware designs |
| 22 | [Specialized domains](22-specialized-domains/README.md) | IoT, media, business/contact/EUC, industry, HPC, quantum, games, location, space |
| 23 | [Projects and reference practice](23-practice-and-reference/README.md) | Ten capstones, fifteen troubleshooting scenarios, career/interview and evidence workflows |

## Start with a concrete result

Run the local HTTP diagnostic, transactional SQL, and duplicate-request exercises in subject 01. Complete account/tooling setup and the small S3/SQS labs if you have an authorized sandbox. Then work through the foundation subjects and build one capstone incrementally.

Each subject's **Coverage mapping** table connects original concepts and service families to the new lessons. The global [coverage register](COVERAGE.md) inventories original headings and the service-map rows so topics can be traced rather than silently dropped. Related reference inventories and templates stay linked to their existing files to avoid contradictory copies.

## Evidence and scope

These files provide runnable local examples, optional AWS procedures, and explicitly labeled worked designs for expensive or enterprise-specific subjects. The writing process does not deploy AWS resources. Verify your account's capabilities, current documentation, permissions, prices, quotas, runtimes/models, and lifecycle before live execution.

Complete a lesson by preserving its verification/failure evidence and cleanup result. Opening every file or executing a sample once is not equivalent to production expertise. Use the [master readiness checklist](../26-master-completion-checklist.md) and [project briefs](../23-projects-and-portfolio.md) to connect lessons to independently demonstrated work.
