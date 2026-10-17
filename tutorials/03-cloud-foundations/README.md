# Cloud foundations tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../03-cloud-foundations.md) · Reviewed 2026-10-18

Work through these in order. The first is a read-only account exercise; the other two produce local designs and runnable simulations. No tutorial requires buying infrastructure.

| Tutorial | Result |
| --- | --- |
| [01 — Map resource scopes and responsibility](01-resource-scopes-and-responsibility.md) | Account/Region/AZ inventory and a responsibility matrix |
| [02 — Turn requirements into architecture](02-requirements-and-architecture.md) | A costed decision record for a document service |
| [03 — Test distributed behavior and recovery](03-distributed-behavior-and-recovery.md) | Retry/idempotency demonstration and failure test plan |

## Coverage mapping

| Source heading and named subtopics | Worked tutorial |
| --- | --- |
| Cloud concepts: scalability, elasticity, availability, durability, reliability, agility, consumption pricing; IaaS, managed platforms, serverless, SaaS | 02, 03 |
| Global infrastructure: Region/AZ, AZ IDs, resource scope, edge, Local Zones, Outposts | 01 |
| Accounts, identity, networks, endpoints; Organizations/management accounts; control and data planes | 01, 03 |
| Shared responsibility: EC2 OS, managed data, security, recovery and operations ownership | 01 |
| Requirements: users, traffic, sensitivity, latency percentiles, availability, RTO/RPO, skills and budget | 02 |
| Distributed behavior: ambiguous success, retries, jitter, idempotency, queues, DLQs, backpressure, state, compatibility, clock skew and fault boundaries | 03 |
| Well-Architected six pillars, cost units, quotas versus capacity | 02 |
| Implementation workflow and architecture decision lab | 02, 03 |
| Read-only resource inventory lab | 01 |
| Common mistakes and mastery outcomes | Verification sections in all three |

Keep evidence locally. Mask account identifiers when publishing portfolio material, and never include session tokens or credentials.
