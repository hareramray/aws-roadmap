# 07 — Object, block, file storage, and recoverable backups

[Roadmap index](README.md) · Prerequisites: [cloud foundations](03-cloud-foundations.md), [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md)

Reviewed: **2026-10-18**. Choose storage from access semantics and recovery requirements. Capacity alone is not enough: latency, throughput, IOPS, concurrency, durability, retention, security, and restore behavior all matter.

## Outcomes

Choose S3, EBS, EFS, or a suitable FSx service; configure private object access; explain versioning, replication, lifecycle, and immutability; and prove that a backup can restore useful application data.

## 1. Storage semantics and selection

| Storage model | Typical services | Best starting uses | Key constraint |
| --- | --- | --- | --- |
| Object | S3 | Uploads, archives, static assets, data lakes | Object/API semantics, not a general POSIX filesystem |
| Block | EBS | EC2 boot/data volumes, supported database storage | Zonal attachment and filesystem/application consistency |
| Local ephemeral block | EC2 instance store | Scratch data, caches, replicated transient work | Data loss on relevant host/instance lifecycle events |
| Shared file | EFS | Linux-oriented NFS sharing | Network, permissions, throughput, file access patterns |
| Specialized file | FSx families | Windows SMB, Lustre, ONTAP, OpenZFS needs | Different protocols, topology, features, and operational choices |

Compare random I/O with sequential throughput, small-file workloads with large objects, hot working set with total data, and single-writer with concurrent access. Measure end-to-end application latency, including encryption, network, client concurrency, and metadata operations.

## 2. S3 object and bucket fundamentals

A general purpose S3 bucket contains objects addressed by keys. Slash-separated keys support useful naming conventions but do not turn S3 into a traditional directory tree. Objects have metadata, optional tags, checksums, storage class, and potentially version IDs.

Learn PUT, GET, HEAD, LIST, COPY, DELETE, multipart upload, presigned access, range reads, and pagination. A multipart upload creates parts before completion; abandoned parts can remain billable, so configure cleanup. An ETag is not a universally valid MD5 checksum, particularly across multipart and encryption behaviors; use supported checksum fields and verify the integrity property you need.

S3 provides strong consistency for successful object writes, overwrites, deletes, and subsequent reads/listing. This does not create multi-object transactions, synchronize cross-Region replicas instantly, or guarantee that a separate CDN cache immediately changes. Applications still need concurrency control for competing writers. [S3 consistency](https://aws.amazon.com/s3/consistency/)

Treat directory buckets and S3 Express One Zone as a distinct advanced branch: their zonal behavior, endpoint and authorization model, naming, and feature support differ from general purpose buckets. Likewise, structured table or vector storage features require their own data-model study; do not apply general purpose bucket assumptions blindly.

## 3. S3 authorization and secure delivery

Begin with Block Public Access, intentional ownership settings, and private access. Prefer policy-based access with ACLs disabled where supported by the workload. Separate bucket-level actions from object actions, and combine IAM, bucket/access-point policies, endpoint policy, and KMS permissions as required.

A presigned URL delegates a particular signed request for a bounded period. It is a bearer capability: anyone who obtains it may be able to use it within its effective validity. Limit operation, object, lifetime, and logging exposure. A URL created using temporary credentials cannot outlive the credentials' useful validity.

For distribution, use CloudFront with a private S3 REST origin and suitable origin access control. A public website endpoint has different security and protocol behavior. Cross-origin resource sharing (CORS) controls browser access behavior; it is not a substitute for authorization and does not prevent nonbrowser clients from making permitted requests.

Choose encryption deliberately: service-managed S3 encryption, SSE-KMS, applicable dual-layer options, or client-side encryption according to requirements. KMS introduces key policy, API usage, grants, recovery dependencies, and cost considerations. Changing default encryption does not automatically transform every existing object's encryption state.

## 4. Versioning, lifecycle, replication, and retention

Versioning retains object variants. A normal delete in a versioned bucket generally adds a delete marker; prior versions can remain recoverable. Permanently deleting a specific version removes that recovery option. Suspending versioning does not erase existing versions or return the bucket to a never-versioned state. [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)

Lifecycle rules can transition storage classes and expire current/noncurrent versions. Account for minimum storage durations, retrieval behavior, small-object overhead, and transition costs before assuming an archive class saves money. A rule for current objects alone may leave all old versions accumulating.

Choose classes by access requirements: frequent access, unpredictable access, infrequent access with retrieval needs, or archive retrieval windows. S3 Intelligent-Tiering can help with uncertain access patterns but still requires understanding eligible objects, monitoring charges, and optional archive tiers. One Zone classes have a different failure scope from multi-AZ classes. [S3 storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

Replication can support locality, separate-account copies, and recovery, but its rules need explicit testing: new versus existing objects, versioning, encrypted objects, delete markers, ownership, and destination permissions. Existing objects may require a batch replication workflow. Replication can propagate unwanted changes; it is not a complete backup strategy.

S3 Object Lock provides write-once/read-many retention and legal holds with governance and compliance behaviors. Once Object Lock is enabled, it cannot be disabled and versioning cannot be suspended for that bucket. Do not enable irreversible retention in routine disposable labs. [Object Lock considerations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html)

## 5. EBS and snapshot design

EBS volumes provide block devices attached to EC2 in the same AZ. General-purpose SSD, provisioned-IOPS SSD, and HDD options serve different I/O patterns. Compare volume performance with instance EBS limits and workload queue depth. Resizing a volume may also require partition and filesystem expansion inside the OS. [EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html)

Multi-Attach is specialized; attaching a volume to multiple instances does not make an ordinary filesystem safe for concurrent writers. The application/filesystem must coordinate access appropriately, and supported configurations matter.

Snapshots capture volume data for recovery and creation of new volumes. Distinguish crash-consistent from application-consistent snapshots. Quiesce writes or use supported application-aware procedures when required. For multi-volume data, coordinate the recovery point across volumes.

Incremental snapshot storage does not mean you must manually retain every older snapshot in a chain for later snapshots to work. AWS manages shared blocks; deleting a snapshot frees only blocks no remaining snapshot requires. Encryption keys must remain available for encrypted restores. Snapshot restore initialization and optional acceleration features affect performance and cost.

## 6. Shared file storage

EFS provides NFS file access, with Regional and One Zone file-system options. Study mount targets, network access, encryption in transit, IAM-based controls, access points, and POSIX permissions. A network allow rule alone does not give a process permission to every file. Throughput/performance modes and lifecycle storage classes must fit the workload. [EFS overview](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)

Choose FSx by filesystem needs rather than treating the family as one interchangeable product:

- **FSx for Windows File Server:** SMB and Windows-oriented integration.
- **FSx for Lustre:** high-throughput parallel file workloads, including suitable S3 integration patterns.
- **FSx for NetApp ONTAP:** ONTAP features and supported multiprotocol storage requirements.
- **FSx for OpenZFS:** managed OpenZFS capabilities for compatible applications.

Study deployment options, backup support, throughput provisioning, access control, maintenance, and restore procedures in the selected family's guide. A POSIX-like mount is not proof that database vendor requirements are satisfied.

## 7. Hybrid movement and storage operations

DataSync can automate supported data movement with verification and scheduling. Storage Gateway supports hybrid access patterns with service-specific local caching or volume/tape semantics. Transfer Family supports managed file-transfer workflows. Choose based on protocols, source access, connectivity, data volume, metadata fidelity, and cutover requirements.

Inventory and S3 Storage Lens can help answer what exists and how it is used. Access logs and CloudTrail data events answer different questions and have different costs. Plan object naming, retention, classification, ownership, and deletion processes before a bucket grows into an unmanaged archive.

## 8. Backup architecture and restore evidence

A backup plan specifies protected resources, schedule, retention, destination, access controls, encryption, restore owner, and verification. AWS Backup centralizes supported resource protection through plans and vaults, but service and Region support vary. Check whether a feature supports continuous backup, cross-account copy, cross-Region copy, and restoration for the exact resource.

Use a separate security boundary when the threat includes compromised production credentials. Protect the destination and its key as carefully as the source. Vault Lock governance and compliance modes have different removal behavior; compliance locks become immutable after the configured grace period, so they can create unavoidable retention costs. Study this as a design exercise first. [AWS Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)

Restore tests must check useful data, not only a job's successful status. Restore to an isolated environment, verify checksums and application queries, measure time, confirm permissions and keys, and clean up. Backups without a tested application recovery procedure are unproven.

## Lab A — Private versioned object recovery

**Type:** sandbox S3 lab with tiny fabricated files.

1. Create a uniquely named general purpose bucket in your chosen Region. Keep it private, use policy-based ownership controls, and enable versioning.
2. Upload `training/notes.txt` containing `version-one`. Record its version ID and checksum.
3. Overwrite it with `version-two`, then retrieve the current object and inspect the versions view.
4. Delete the object without specifying a version. Observe the delete marker and recover the earlier content using an explicit version or the console restore/copy workflow.
5. Draft a lifecycle rule for noncurrent versions and incomplete multipart uploads. Review its scope before applying it to lab-only data.

**Verify:** compare restored content with the original; explain why listing current objects differs from listing versions. Verify anonymous access fails. **Cleanup/cost:** permanently remove every lab version and delete marker, then delete the bucket. Normal recursive deletion of only current objects may leave versions behind. Storage, requests, KMS if chosen, and optional telemetry can incur charges. Do not enable Object Lock for this lab.

## Lab B — Backup recovery plan and optional EBS restore

**Baseline:** write a recovery procedure for a small application's database and uploads. Define RPO/RTO, keys, restore destination, validation queries, and access permissions. No AWS deployment is required.

**Optional deployment:** attach a small encrypted data volume to a disposable EC2 instance; write dummy files and hashes; flush writes; create a snapshot; restore a new volume in the intended AZ; mount it separately without overwriting the original. Use [module 06](06-compute-and-load-balancing.md) for secure administration.

**Verify:** file hashes match and the restored data is usable. Record elapsed restore time and remaining dependencies. **Cleanup:** unmount/detach and remove only the dedicated test volumes and snapshot, then the disposable instance and associated resources. Snapshot storage continues after instance deletion. Never format a device until its volume identity is confirmed.

## Mastery checklist

- [ ] Choose object, block, or file semantics from application requirements.
- [ ] Explain strong S3 consistency without implying multi-object transactions.
- [ ] Recover a versioned object and completely empty a test bucket.
- [ ] Distinguish replication, versioning, backup, and immutable retention.
- [ ] Explain application-consistent recovery and encryption-key dependency.
- [ ] Verify restored data and account for retained storage costs.

Further references: [S3 guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/), [AWS Backup guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/), [FSx documentation](https://docs.aws.amazon.com/fsx/), [DataSync guide](https://docs.aws.amazon.com/datasync/latest/userguide/).
