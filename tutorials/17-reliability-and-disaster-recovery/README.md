# Reliability and disaster recovery tutorials

[All tutorials](../README.md) · [Source roadmap](../../17-reliability-and-disaster-recovery.md)

| Tutorial | Deliverable |
| --- | --- |
| [01 — Restore data and prove correctness](01-restore-and-validate.md) | A runnable local restore, integrity evidence, measured recovery, and an AWS Backup extension |
| [02 — Design and rehearse failover](02-failover-and-game-day.md) | Capacity model, data authority rules, recovery timeline, and a bounded game-day plan |

## Coverage mapping

| Original heading or branch | Worked coverage |
| --- | --- |
| Define what must survive | 01 business records/RPO/RTO; 02 failure-scope matrix |
| Resilience before regional recovery | 02 surviving capacity, retries, idempotency, backpressure, cache warming |
| DR strategies | 02 backup/restore, pilot light, warm standby, active/active comparison |
| Protected usable backups | 01 application consistency, AWS Backup plans/vaults, copies, Vault Lock, air-gapped vaults, restore testing |
| Multi-Region data | 02 RDS/Aurora/Global Database, DynamoDB modes, S3 replication, queues/caches, split brain |
| Whole dependency chain | 02 IAM/KMS/artifacts/quotas, DNS, Global Accelerator, ARC, Elastic Disaster Recovery, failback |
| Restore lab | 01 full runnable exercise with integrity and business checks |
| Game days | 02 FIS scope/stop conditions, Resilience Hub assessment, human decision process |
| Pitfalls and mastery | Acceptance criteria and failed scenarios in both tutorials |

Run the local exercise first. Live backup copies, restored databases, replication, DR servers, and failover infrastructure are optional paid extensions. Do not enable irreversible retention or fault injection against shared resources. Record whether every result is measured, simulated, or estimated.
