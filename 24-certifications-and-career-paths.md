# 24 — Certifications and career paths

[Roadmap index](README.md) · Pair with [projects](23-projects-and-portfolio.md)

**Certification information reviewed:** 18 October 2026. Exam names, codes, language availability, beta periods, fees, and retirement dates can change. Verify the current official exam guide and booking page before buying materials or scheduling. This chapter recommends learning sequences; it does not promise exam success or employment.

## 1. Choose a role before choosing an exam

Certification can organize study and demonstrate familiarity with a defined scope. It does not replace experience diagnosing a broken system, securing a deployment, restoring data, or explaining tradeoffs. Select an exam that reinforces the role you want, then build projects that expose the gaps exam questions cannot measure.

| Direction | Useful learning sequence | Practical evidence |
|---|---|---|
| New to IT/cloud | Prerequisites → cloud foundations → optional Cloud Practitioner | Secure account, simple workload, basic troubleshooting |
| Architecture | Core services → Solutions Architect Associate scope → real design work → Professional scope | Design records, resilient app, migration and DR exercises |
| Development | Programming → IAM/data/serverless → Developer Associate scope | Tested API, idempotent worker, automated deployment |
| Operations | Linux/networking → monitoring/recovery → CloudOps Associate scope | Runbooks, incidents, backups, cost investigations |
| DevOps/platform | Delivery/IaC/containers → operations experience → DevOps Professional scope | Reusable infrastructure, release automation, rollback and reliability evidence |
| Security | IAM/networking/security → operational threat response → Security Specialty scope | Least-privilege review, detection, incident and recovery evidence |
| Data engineering | SQL/data modeling → analytics pipelines → Data Engineer Associate scope | Quality checks, lineage, backfill, stream processing |
| ML/AI engineering | Software/data/ML fundamentals → ML Engineer Associate or Generative AI Developer Professional scope as appropriate | Reproducible training or evaluated AI system, deployment and monitoring |
| Networking | Deep networking and hybrid design → relevant architecture/security study | Routing, DNS, redundant hybrid links, packet-path diagnosis |

These are suggested routes, not mandatory AWS prerequisites. You do not need to earn every foundational and associate certification before studying a more advanced scope. You do need the underlying practical competence.

## 2. Certification inventory

The official guide index currently lists the following families/codes. Descriptions in the final column are learning directions rather than full exam outlines. [AWS Certification Exam Guides](https://docs.aws.amazon.com/aws-certification/latest/examguides/)

| Level | Certification / code | Study emphasis |
|---|---|---|
| Business | AI Business Strategist — AIB-C01 | Business outcomes, governance, adoption, responsible AI |
| Foundational | Cloud Practitioner — CLF-C02 | Cloud concepts, security responsibilities, services, billing |
| Foundational | AI Practitioner — AIF-C01 | AI/ML concepts, generative AI, responsible use |
| Associate | Solutions Architect — SAA-C03 | Secure, resilient, performant, cost-aware design |
| Associate | Developer — DVA-C02 | Application development, security, deployment, debugging |
| Associate | CloudOps Engineer — SOA-C03 | Operations, monitoring, resilience, networking and cost |
| Associate | Data Engineer — DEA-C01 | Ingestion, transformation, stores, operations and governance |
| Associate | Machine Learning Engineer — MLA-C02 beta; MLA-C01 language transition | Operational ML/AI lifecycle; verify the version for your language |
| Professional | Solutions Architect — SAP-C02 | Complex organizations, architectures, modernization |
| Professional | DevOps Engineer — DOP-C02 | Delivery automation, operations, resilience and governance |
| Professional | Generative AI Developer — AIP-C01 | Advanced generative AI application implementation |
| Specialty | Security — SCS-C03 | Security design, data protection, detection and response |
| Specialty | Advanced Networking — ANS-C01, retirement scheduled | Advanced cloud/hybrid network design and operations |

## 3. Changes that make older roadmaps misleading

CloudOps Engineer Associate is the current name associated with SOA-C03; older materials may use SysOps Administrator Associate. Existing older credentials do not simply change names retroactively. Use the current exam outline when selecting preparation material. [CloudOps certification](https://aws.amazon.com/certification/certified-cloudops-engineer-associate/)

The Machine Learning Engineer Associate transition is language-sensitive. As of this review, MLA-C02 is in English beta; the English MLA-C01 last-test date was 28 September 2026, while the older version remains available in specified other languages during the transition. Check the current page for the exact language and availability you need. [Machine Learning Engineer certification](https://aws.amazon.com/certification/certified-machine-learning-engineer-associate/)

Advanced Networking Specialty has an announced last-test date of 31 December 2026. Do not make a long-term study plan that assumes indefinite availability of that exam. The networking skills remain useful regardless of the credential's lifecycle. [Advanced Networking certification](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)

Machine Learning Specialty retired on 31 March 2026. Historical materials can still teach concepts, but it should not be presented as a currently bookable target. AWS's AI portfolio now includes other role-specific options. [AWS AI certification updates](https://aws.amazon.com/blogs/training-and-certification/big-news-aws-expands-ai-certification-portfolio-and-updates-security-certification/)

For Security Specialty, use the current SCS-C03 guide rather than assuming that older SCS-C02 materials map perfectly. [Security certification](https://aws.amazon.com/certification/certified-security-specialty/)

## 4. Build an exam preparation matrix

Download the official guide for the exact code. Copy its domain names and weights into your private study tracker rather than relying on a third-party summary. For each domain, record required concepts, related roadmap chapters, one practical exercise, your confidence, and mistakes from practice questions.

Study the “out of scope” and target-candidate sections too. An exam is not the complete AWS platform, and an exam's exclusions should not be interpreted as a reason to omit necessary production skills.

| Tracker field | Example |
|---|---|
| Objective | Diagnose access failures across identity and resource policies |
| Roadmap | Chapters 04, 09, 19, 31 |
| Practical evidence | Authorized and denied requests with an explanation of policy evaluation |
| Gap | Confusing a boundary with a permission grant |
| Next action | Read the policy-evaluation guide and repeat the constrained-role exercise |

## 5. Preparation cycle

1. Take an official sample question set to identify gaps, without treating the score as a final prediction.
2. Learn the underlying service behavior from official documentation and this roadmap.
3. Build a small exercise for concepts that remain abstract.
4. Answer new scenario questions and explain why each alternative is unsuitable.
5. Revisit errors by category: missing knowledge, misunderstood requirement, overlooked constraint, or time pressure.
6. Use an official practice exam when ready, then remediate remaining weaknesses.
7. Verify current exam policies, identity requirements, language, accommodations, and testing setup before booking.

Avoid recalled-question dumps. They undermine learning and can violate exam agreements. Prefer legitimate official preparation and questions that require reasoning. Memorizing a phrase such as “use Multi-AZ” is insufficient if you cannot distinguish availability, read scaling, and disaster recovery.

## 6. Readiness criteria

You should be able to explain the major services in the target scope, choose between plausible designs, identify cost/security/reliability constraints, and work through unfamiliar scenarios calmly. Repeated high scores on the same memorized questions are weak evidence; performance on fresh material plus practical demonstrations is more useful.

Before considering professional-level preparation complete, review multi-account governance, cross-region failure, migration dependencies, deployment rollback, operational processes, and cost tradeoffs. These topics become difficult when several valid requirements conflict.

## 7. Turn learning into career evidence

Write a short project case study using this structure: user problem, constraints, architecture, implementation, measurable tests, failure encountered, recovery, costs, and next improvement. Use precise claims: “recovered a synthetic database in 18 minutes during one exercise” conveys more than “implemented disaster recovery.”

For interviews, prepare a five-minute explanation and a deeper technical walkthrough of each project. Expect questions about what you would change at ten times the load, how permissions are scoped, how data is recovered, and how you would detect a failed deployment.

Create a skills matrix matching the role description. Separate independently demonstrated skills from guided labs and topics you have only read about. Certifications can appear alongside this evidence; they should not be the only evidence.

## Official preparation sources

- [AWS Certification](https://aws.amazon.com/certification/) — current portfolio and policies.
- [Exam guides](https://docs.aws.amazon.com/aws-certification/latest/examguides/) — exact scope and exam codes.
- [AWS Skill Builder](https://skillbuilder.aws/) — official courses and exam preparation; check free versus paid access.
- [Certification FAQs](https://aws.amazon.com/certification/faqs/) — scheduling and policy questions.

- [ ] I chose a role-aligned target and checked its current lifecycle.
- [ ] My study tracker follows the exact exam guide/version.
- [ ] I can demonstrate practical work beyond answering questions.
- [ ] I checked current booking requirements instead of relying on an old tutorial.
