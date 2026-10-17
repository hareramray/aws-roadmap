# 00 — Learning plan and role-based routes

[Roadmap index](README.md) · Next: [Prerequisites](01-prerequisites.md)

## Learning target

Your target is to design, build, secure, operate, recover, and explain an AWS workload. Recognizing service names is only the first step. A useful proficiency scale is:

| Level | Evidence |
|---|---|
| Recognize | Explain the problem a service solves and name an alternative |
| Build | Implement a small working example from documentation |
| Explain | Justify configuration, permissions, cost, and failure behavior |
| Operate | Diagnose a failure and restore service using a runbook |
| Design | Compare competing architectures against measurable requirements |
| Improve | Use production-like evidence to reduce risk, latency, or cost |

Move forward when you can demonstrate the required level, not simply when a video ends. Revisit earlier material during projects: IAM, networking, data modeling, and observability become clearer when they interact.

## Phase 0: technical readiness

Complete [01](01-prerequisites.md) and [02](02-account-and-tooling.md). You should be able to navigate a terminal, edit JSON/YAML, use Git, explain an HTTP request, query a small SQL database, and run a script that handles an error. Establish a sandbox, identity checks, billing alerts, and a cleanup habit before provisioning infrastructure.

**Exit evidence:** a local HTTP application, a Git repository without secrets, a documented account/region, and successful `sts get-caller-identity` output recorded privately.

## Phase 1: the cloud foundation

Study chapters 03–09. Learn the request path in this order: identity authorizes an action; networking carries traffic; compute runs code; storage and databases preserve state; security and monitoring protect and explain the system. Build a private object store and a simple compute workload before adding high availability.

**Exit evidence:** diagram a two-AZ application, explain each route and permission, restore one deleted test object or data set, and diagnose an intentional access denial.

## Phase 2: application delivery

Study chapters 10–13. Build one API with asynchronous processing, deploy a container, codify infrastructure, and automate delivery. Learn why retries require idempotency and why a successful deployment is different from a healthy service.

**Exit evidence:** a repeatable deployment, an automated check, a dashboard, a dead-letter queue or equivalent failure path, and a demonstrated rollback.

## Phase 3: data and intelligent applications

Study chapters 14–15. Everyone should understand ingestion, data quality, access control, and the difference between operational and analytical stores. Go deeper into data engineering or AI only if your role requires it. A working model call is not an evaluated application.

**Exit evidence:** a small reproducible data pipeline or evaluated AI application with a data contract, access rules, cost estimate, and a deletion procedure.

## Phase 4: production engineering

Study chapters 16–20. Add service-level objectives, useful alerts, recovery procedures, account guardrails, and financial accountability. Compare your original architecture with the evidence collected during load and failure tests.

**Exit evidence:** an incident exercise, a measured restore, a cost breakdown, an architecture decision record, and a prioritized improvement plan.

## Phase 5: specialization and portfolio

Study chapters 21–22 and choose projects from chapter 23. Add role-specific depth while retaining the core foundation. Use chapters 24–26 to evaluate readiness, and keep a source/update log from chapter 28.

**Exit evidence:** two or three finished projects with understandable documentation, reproducible setup, security decisions, test results, cost assumptions, and cleanup steps.

## A 24-week core schedule

This is a planning example for roughly 8–12 focused hours per week, not a guarantee of mastery. A complete beginner may need longer; deep specialization takes additional time.

| Weeks | Focus | Deliverable |
|---|---|---|
| 1–2 | Terminal, Linux, Git, HTTP, SQL, Python; account setup | Local app and secure tooling checklist |
| 3 | Cloud model and IAM | Access diagram and least-privilege exercise |
| 4–5 | VPC, routing, DNS, EC2, load balancing | Request-path diagram and compute lab |
| 6 | S3, EBS, EFS, backup | Storage decision record and restore evidence |
| 7–8 | RDS, DynamoDB, caching, security | Data model, threat model, access tests |
| 9–10 | Lambda, APIs, queues, events, workflows | API with asynchronous job processing |
| 11–12 | Containers, ECS, Kubernetes fundamentals | Container deployment and health checks |
| 13–14 | IaC and CI/CD | Reproducible environment and rollback |
| 15–16 | Data pipeline fundamentals; optional AI introduction | Data-quality report or AI evaluation set |
| 17 | Metrics, logs, traces, operational runbooks | Dashboard and actionable alert |
| 18 | Backup, availability, DR | Restore report with measured RTO/RPO |
| 19 | Architecture and governance | Design review and account-boundary proposal |
| 20 | FinOps and performance | Cost model and bottleneck experiment |
| 21 | Migration, hybrid, specialist survey | Migration or specialization design |
| 22–24 | Capstone, failure drills, review | Finished portfolio project and readiness assessment |

For a 12-week accelerated route, combine pairs of weeks only if you already know the prerequisites and can spend more time. Do not remove recovery, security, or cleanup to meet a calendar. For a 36–48-week route, keep the same order and use extra weeks for repeated projects and deeper specialization.

## Weekly study rhythm

- **Concept session:** read a chapter and its key official documentation; write five questions.
- **Build session:** implement one small piece and record exact assumptions.
- **Diagnostic session:** break one controlled property, predict the symptom, collect evidence, and repair it.
- **Review session:** explain the tradeoffs aloud, update notes, and clean up resources.

A practical allocation is 30% reading, 50% building/debugging, and 20% review/documentation. Adjust this to your gaps; it is a study heuristic, not a measured industry rule.

## Role routes

| Target role | Deep focus | Evidence to prioritize |
|---|---|---|
| Cloud support / CloudOps | 04–09, 16–17, 20–21, 31 | Troubleshooting, patching, access diagnosis, recovery, customer-facing incident notes |
| Solutions architect | 03–10, 17–22 | Requirement discovery, tradeoffs, estimates, diagrams, migration and governance |
| Cloud developer | 04, 08, 10–13, 16 | Secure APIs, SDK usage, testing, idempotency, deployment and debugging |
| DevOps / platform / SRE | 05–06, 09–13, 16–20 | Reusable infrastructure, pipelines, reliability, access controls, operational ownership |
| Security engineer | 04–05, 09, 16–19, 31 | Threat models, prevention/detection, evidence, incident response and identity analysis |
| Data engineer | 07–08, 10, 12–14, 16, 19–20 | Data contracts, quality, lineage, partitioning, streaming and recovery |
| ML / AI engineer | 07–08, 10–15, 16, 20 | Reproducible experiments, evaluations, secure inference, monitoring and unit economics |
| Network / hybrid engineer | 05, 09, 17–19, 21 | Addressing, routes, DNS, redundant connectivity and failure isolation |

These routes select depth; they do not remove the common foundation. An AI engineer still needs IAM and networking. An architect still needs enough hands-on experience to recognize infeasible designs.

## Avoid common learning traps

Do not collect certifications without building. Do not copy large templates you cannot explain. Avoid changing several things during a failure investigation because you will lose causal evidence. Do not assume that a service is suitable merely because it is managed. Managed services still require data modeling, permissions, capacity decisions, recovery procedures, and cost control.

When stuck, reduce the system to the smallest reproducing case. Write what should happen, what actually happens, and the boundary at which they diverge. Use [the troubleshooting playbook](31-troubleshooting-playbook.md) and record the lesson in your journal.

## Progress record

For every chapter, track status (`not started`, `reading`, `lab`, `verified`, `revisit`), date, evidence link, confidence, and next gap. Score your ability from 0–3: unknown, explain, build, operate. Mark a core chapter complete only after reaching at least explain/build, and finish the capstone at operate level.

- [ ] I chose one role route and a realistic weekly schedule.
- [ ] I know which skills I already have and which prerequisites need work.
- [ ] I use evidence-based exit gates rather than just study hours.
- [ ] I reserve time and budget for testing, recovery, and cleanup.
- [ ] I can explain my next project in terms of a user problem.
