# Storage and backup tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../07-storage-and-backup.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — Private S3, versions and storage classes](01-private-s3-and-version-recovery.md) | Object recovery, access tests and lifecycle design |
| [02 — Block and shared file storage](02-block-and-shared-file-storage.md) | EBS restore plus EFS/FSx protocol/performance designs |
| [03 — Backups, replication and hybrid movement](03-backup-replication-and-transfer.md) | Restore evidence, protected-copy design and migration reconciliation |

## Coverage mapping

| Source heading and named topics | Tutorial |
| --- | --- |
| Storage semantics: object/block/file/ephemeral, latency/IOPS/throughput/concurrency | 01, 02 |
| S3 fundamentals: keys, metadata/tags/checksums, operations/pagination/range/multipart, strong consistency | 01 |
| S3 security/delivery: ownership/BPA/policies, presigned access/CORS, CloudFront OAC, encryption | 01 |
| Versioning/lifecycle/classes/Intelligent-Tiering/One Zone/archive/replication/Object Lock | 01, 03 |
| EBS: types, attachment/Multi-Attach, limits/resizing, snapshot consistency/incremental/restore initialization | 02 |
| EFS: mount targets, NFS, access points/IAM/POSIX, encryption, throughput; FSx four families | 02 |
| Hybrid movement: DataSync, Storage Gateway, Transfer Family; Inventory/Storage Lens/logs | 03 |
| Backup: plans/vaults, cross-account/Region, key dependency, Vault Lock, verified recovery | 03 |
| Private version recovery lab, optional EBS restore lab | 01, 02 |
| Coverage-map extensions: directory buckets/Express, S3 Tables/Vectors | 01 |
| Mastery and failure scenarios | Verification in all |
