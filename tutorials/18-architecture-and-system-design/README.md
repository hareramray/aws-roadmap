# Architecture and system design tutorials

[All tutorials](../README.md) · [Source roadmap](../../18-architecture-and-system-design.md)

| Tutorial | Result |
| --- | --- |
| [01 — Size and select an order platform](01-size-and-select-an-order-platform.md) | Requirements, capacity model, service comparisons, cost inputs, and an ADR |
| [02 — Prove correctness and review the architecture](02-correctness-and-architecture-review.md) | A runnable transaction/outbox example, tenant tests, failure design, and six-pillar review |

## Coverage mapping

| Original heading | Worked material |
| --- | --- |
| Requirements | 01 measurable functional/quality requirements and assumptions |
| Six Well-Architected pillars | 02 evidence-based review and prioritized improvements |
| Estimate scale | 01 traffic/concurrency/storage/network calculation and sensitivity test |
| Compute and boundaries | 01 EC2/ECS/EKS/Lambda/managed service comparison; modular monolith decision |
| Data and consistency | 02 invariants, relational/key-value/search/object choices, caches, transactions |
| Asynchronous flows | 02 outbox, duplicate effects, saga, queue/topic/event/stream decision |
| Cross-cutting concerns | 02 tenant isolation, threat model, authorization, compatibility, degradation |
| Order-platform exercise | Both tutorials form one complete worked design |
| ADR template | 01 completed decision with revisit conditions |
| Pitfalls and mastery | Failure tests, acceptance tables, cost/cleanup in both |

The baseline is local design and Python/SQLite work. AWS deployments are optional. Use [database tutorials](../08-databases-and-caching/README.md) and [serverless tutorials](../10-serverless-and-integration/README.md) when implementing the selected architecture. A design is accepted only when its assumptions, failure behavior, ownership, and evidence can be explained.
