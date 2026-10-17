# Tutorial 07.2 — Restore a block volume and choose a shared filesystem

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../07-storage-and-backup.md)

**Mode:** optional EBS sandbox lab; EFS/FSx branches are worked designs unless deliberately deployed. **Prerequisites:** disposable Linux EC2 instance with Session Manager. **Result:** verified restored files and a protocol/performance decision. Reviewed 2026-10-18.

## 1. Select semantics before a product

Compare an application boot disk, shared Linux home directories, a Windows SMB application, and a parallel scientific scratch dataset. The boot disk needs block storage; the sharing workloads need filesystem semantics/protocols; reproducible scratch data may tolerate ephemeral storage.

Record random IOPS, sequential throughput, latency, metadata operations, concurrency and durability for each. A million tiny files and one large sequential object have different bottlenecks. An S3 mount adapter does not automatically make object storage equivalent to a general POSIX filesystem.

## 2. Attach a dedicated EBS data volume

Create a small encrypted general-purpose SSD volume in the **same AZ** as the disposable instance. Name/tag it clearly and record its volume ID. Attach it. Inside the Linux instance use `lsblk` and device metadata to identify the new device by volume identity; Nitro device names can differ from the requested attachment name.

Only after confirming it is the newly created empty volume, create a filesystem and mount it at a dedicated path. The following is **illustrative Bash**, not copy-and-run with an unverified device:

```bash
# Replace /dev/CONFIRMED_LAB_DEVICE only after checking the EBS volume ID.
sudo mkfs -t xfs /dev/CONFIRMED_LAB_DEVICE
sudo mkdir -p /mnt/training-data
sudo mount /dev/CONFIRMED_LAB_DEVICE /mnt/training-data
printf '%s\n' 'training-record-001' | sudo tee /mnt/training-data/record.txt
sha256sum /mnt/training-data/record.txt
```

Never format the root disk or an existing data volume. For an inexperienced learner, use the EC2/EBS documentation's device-identification procedure and stop if identity is ambiguous.

## 3. Snapshot and verify restoration

Flush writes and unmount this dedicated filesystem to make this simple file exercise's consistency explicit. Create a snapshot of the data volume through the console and wait for completion. Record snapshot ID, encryption key reference and timestamp.

Create a **new** volume from that snapshot in the intended AZ and attach it separately. Mount the restored filesystem read-only using suitable filesystem-specific options; cloned XFS filesystems can require `nouuid` when both copies are attached. Do not run `mkfs` on the restored volume, which would erase what you are trying to verify.

Read `record.txt` and compare its SHA-256 hash with the original. Record elapsed restore/validation time. A completed snapshot job alone is insufficient evidence; your test proves the restored content is useful.

For a real database, unmounting a file test is not a substitute for application-aware backup. Define quiescing/flush and multi-volume coordination requirements according to the database. Crash consistency and application consistency answer different recovery questions.

## 4. Evaluate EBS performance and lifecycle

Compare general-purpose SSD with provisioned-IOPS SSD for latency-sensitive random I/O, and throughput/cold HDD options for supported sequential access patterns. Consult current minimum sizes and performance limits before attempting a configuration. The instance's EBS bandwidth can constrain an otherwise powerful volume.

For resizing, write the two-stage workflow: modify the volume, then extend the partition/filesystem if needed. More capacity does not always mean the OS immediately sees a larger filesystem. Benchmark after the supported change completes.

Multi-Attach is specialized: multiple attachments do not make a normal filesystem safe for concurrent writers. Design a cluster-aware coordination mechanism before considering it. Instance store can serve scratch/cache data but must not hold the only authoritative record through stop/host loss.

EBS snapshots are incrementally stored, but AWS manages shared block dependencies. You do not manually retain every old snapshot in a chain for newer snapshots to work. Restored-volume initialization and optional acceleration features affect first-read performance and cost; measure the actual application recovery path.

## 5. Design EFS for two Linux clients

Use the requirement “two application instances share uploaded working files.” Define a Regional EFS filesystem unless the documented risk justifies One Zone, mount targets for the intended AZs, and an EFS group allowing NFS from the application group. Choose a supported encrypted mount method.

Define an access point rooted at `/portal` with a deliberate POSIX user/group and creation permissions. Map two hypothetical processes: UID 1001 should write the application directory, while an unrelated identity should not. IAM authorization, network access and POSIX permissions are separate controls; a successful TCP connection does not grant all file permissions.

If optionally deploying, create a dummy file from client A and read it from client B. Test a denied directory access using an intentionally unprivileged process. Review throughput/performance modes and lifecycle classes against measured metadata/data traffic. EFS is not the ordinary supported Windows SMB choice.

## 6. Work four FSx selection cases

For the Windows application, choose an FSx for Windows File Server evaluation: specify SMB, directory integration, shares, user ACLs and backup. Test conceptually that a user without the share/file permission cannot read the data even when network routing works.

For tightly coupled simulation output, evaluate FSx for Lustre: stage a synthetic input dataset from S3, define parallel clients, run/verify an output checksum and return durable output to its authoritative location. Scratch and persistent deployment choices have different recovery implications.

For a company requiring ONTAP features and supported multiple protocols, evaluate FSx for NetApp ONTAP with volumes, access, snapshots/replication and capacity tiers. For an application that depends on OpenZFS behavior, evaluate FSx for OpenZFS with exports, snapshots/clones and supported client semantics. Do not replace a vendor-certified database filesystem merely because both products expose a mount path.

The deliverable is four short configurations and rejection reasons. Expensive filesystems/directories are not required for this introductory comparison.

## Verify, troubleshoot, and clean up

Pass with matching restored hashes and explanations for the four protocol cases. If a mount fails, check device identity or DNS/mount target, network group, filesystem type, encryption mount options and user permissions. If throughput disappoints, inspect client concurrency, metadata operations, storage and instance limits together.

**Cleanup/cost:** unmount and detach dedicated volumes, then remove original/restored volumes and snapshots when no longer needed. Remove any optional EFS mount targets/filesystems or FSx filesystems and retained backups created solely for this lab. Check lifecycle-created backups separately. Never delete shared directories or keys. Storage, provisioned performance, snapshots, throughput and transfer can charge after compute stops.

Sources: [EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html), [EFS](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html), [FSx documentation](https://docs.aws.amazon.com/fsx/).
