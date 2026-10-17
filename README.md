# AWS roadmap: from first principles to production systems

**Audience:** a beginner who wants a complete learning path, or an engineer filling gaps across AWS.  
**Review date:** 18 October 2026.  
**Format:** separate, linked Markdown chapters with explanations, practical work, checks, and official references.

Start with [the learning plan](00-learning-plan.md), complete [prerequisites](01-prerequisites.md), and set up [your account and tools](02-account-and-tooling.md). Then follow the numbered technical chapters. Use the projects to prove that you can combine services into a working system.

For step-by-step implementation, open the [tutorial collection](tutorials/README.md). It covers the entire roadmap in subject folders, including an 18-part AI/ML sequence, worked examples, verification, troubleshooting, and cleanup. Use its [coverage register](tutorials/COVERAGE.md) to trace original topics to tutorials.

## What this roadmap covers

This is a broad curriculum covering the main AWS disciplines and a discovery map for specialized services. Core topics receive detailed treatment; specialist topics include what to learn, when they matter, and where to go deeper. AWS changes continuously, so a finite guide cannot contain every API, regional exception, new launch, or service limit. The [service coverage map](30-service-coverage-map.md) makes the breadth explicit, and the [resource guide](28-official-resources-and-updates.md) explains how to close workload-specific gaps.

Do not try to memorize a product catalog. For every service, learn its purpose, security boundary, failure behavior, scaling model, cost drivers, alternatives, and operating procedures.

## Curriculum

| Stage | Chapter | What you should be able to do |
|---|---|---|
| Plan | [00 — Learning plan](00-learning-plan.md) | Choose a pace and role, track evidence, and sequence the work |
| Prepare | [01 — Prerequisites](01-prerequisites.md) | Work with Linux, networking, Git, code, SQL, and distributed-system basics |
| Prepare | [02 — Account and tooling](02-account-and-tooling.md) | Establish secure access, billing visibility, and a usable local environment |
| Foundations | [03 — Cloud foundations](03-cloud-foundations.md) | Explain regions, availability zones, shared responsibility, and cloud economics |
| Foundations | [04 — IAM and identity](04-iam-and-identity.md) | Design human and workload access with temporary credentials and least privilege |
| Foundations | [05 — Networking and DNS](05-networking-and-dns.md) | Trace requests through VPCs, routes, DNS, firewalls, private endpoints, and hybrid networks |
| Foundations | [06 — Compute and load balancing](06-compute-and-load-balancing.md) | Run, scale, patch, and troubleshoot EC2-based applications |
| Foundations | [07 — Storage and backup](07-storage-and-backup.md) | Select object, block, file, archival, and backup approaches |
| Foundations | [08 — Databases and caching](08-databases-and-caching.md) | Model data and choose transactional, key-value, document, graph, search, and cache systems |
| Foundations | [09 — Security and compliance](09-security-and-compliance.md) | Protect data, detect threats, investigate findings, and produce control evidence |
| Build | [10 — Serverless and integration](10-serverless-and-integration.md) | Build APIs and resilient event-driven workflows |
| Build | [11 — Containers and Kubernetes](11-containers-and-kubernetes.md) | Deploy secure container workloads with ECS and EKS |
| Build | [12 — Infrastructure as code](12-infrastructure-as-code.md) | Reproduce environments and manage infrastructure changes safely |
| Build | [13 — DevOps and delivery](13-devops-and-delivery.md) | Build, test, release, observe, and roll back applications |
| Data | [14 — Data engineering and analytics](14-data-engineering-and-analytics.md) | Build governed batch and streaming data pipelines |
| AI | [15 — AI, ML, and generative AI](15-ai-ml-and-generative-ai.md) | Train or integrate models, evaluate outputs, and operate AI systems |
| Operate | [16 — Observability and operations](16-observability-and-operations.md) | Use telemetry to diagnose issues and run services reliably |
| Operate | [17 — Reliability and disaster recovery](17-reliability-and-disaster-recovery.md) | Define recovery objectives and demonstrate recovery under failure |
| Design | [18 — Architecture and system design](18-architecture-and-system-design.md) | Turn requirements into defensible technical tradeoffs |
| Scale | [19 — Organizations and governance](19-organizations-and-governance.md) | Establish account boundaries, guardrails, and centralized governance |
| Optimize | [20 — Cost, FinOps, and performance](20-cost-finops-and-performance.md) | Explain a bill, optimize unit costs, and tune bottlenecks |
| Extend | [21 — Migration, hybrid, and edge](21-migration-hybrid-and-edge.md) | Assess, migrate, cut over, and operate hybrid workloads |
| Specialize | [22 — Specialized domains](22-specialized-domains.md) | Explore IoT, media, HPC, end-user computing, industry services, and more |

## Practice and reference files

| File | Purpose |
|---|---|
| [23 — Projects and portfolio](23-projects-and-portfolio.md) | Ten substantial project briefs with acceptance criteria and failure drills |
| [24 — Certifications and career paths](24-certifications-and-career-paths.md) | Role-based preparation and current exam lifecycle guidance |
| [25 — Interview and design questions](25-interview-and-design-questions.md) | Technical questions, answer expectations, and design exercises |
| [26 — Master completion checklist](26-master-completion-checklist.md) | Evidence-based gates for foundations, delivery, operations, and specialization |
| [27 — Glossary and mental models](27-glossary-and-mental-models.md) | Terms, distinctions, and common misconceptions |
| [28 — Official resources and updates](28-official-resources-and-updates.md) | Source map and a repeatable update process |
| [29 — CLI and SDK reference](29-cli-and-sdk-reference.md) | PowerShell-first command examples and SDK patterns |
| [30 — Service coverage map](30-service-coverage-map.md) | Service families, priority, and links back to the curriculum |
| [31 — Troubleshooting playbook](31-troubleshooting-playbook.md) | Diagnostic sequences for common cloud failures |
| [32 — Engineering templates](32-engineering-templates.md) | Reusable design, lab, incident, cost, threat-model, and runbook templates |
| [33 — Guided starter labs](33-guided-starter-labs.md) | Two worked CLI labs with validation and cleanup |

## How to use each chapter

1. Read the prerequisite chapters and explain the key ideas without notes.
2. Draw the data path and trust boundaries before deploying anything.
3. Estimate costs and identify the exact account, region, and resource names.
4. Implement a small version; preserve configuration in version control.
5. Verify both successful behavior and an intentional, reversible failure.
6. Record telemetry and explain the root cause, rather than keeping only screenshots.
7. Remove temporary resources and check for retained storage, snapshots, endpoints, and logs.
8. Complete the mastery checklist using evidence from your own work.

The labs are learning instructions, not a claim that infrastructure has already been deployed or tested in your AWS account. Example commands require your own authorized sandbox and suitable permissions. Cost estimates must use the current prices and your region; no chapter promises that a deployment is free.

## Suggested first session

- Read chapters 00–02 and create a learning journal using chapter 32.
- Verify your CLI identity and region, then inspect account billing and your plan.
- Complete the private S3 lab in chapter 33.
- Explain why the bucket is private, how you authenticated, what requests could cost, and what remains after cleanup.
- Continue with cloud foundations, IAM, and networking before building larger projects.

## Currency and sources

Official AWS references are provided within chapters. Service availability, lifecycle, pricing, quotas, account-plan restrictions, and exam versions must be checked before use. The [AWS documentation portal](https://docs.aws.amazon.com/) and [AWS service overview](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/introduction.html) are the starting points for new or omitted workload-specific features.
