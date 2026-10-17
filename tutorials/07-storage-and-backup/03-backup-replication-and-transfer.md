# Tutorial 07.3 — Prove recovery and design protected data movement

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../07-storage-and-backup.md)

**Mode:** recovery design with optional small backup/restore; enterprise replication and immutable retention are design-only. **Result:** a restore report and reconciliation plan. Reviewed 2026-10-18.

## 1. Define what must be recovered

The portal has original documents, a metadata database, derived thumbnails and configuration. Assign RPO/RTO independently: originals/metadata require recovery, thumbnails can be regenerated, and configuration should be reproducible from reviewed definitions.

Write a manifest of five synthetic files with ID, size and checksum, plus five matching metadata rows. Deliberately create one orphan file and one metadata row referencing a missing file. A useful restore test must detect both, not merely show that a bucket and database exist.

Set an exercise RPO of one hour and RTO of four hours. Identify backup frequency, retention and restore order. Access to a decryption key, database credentials, network and application code are recovery dependencies too.

## 2. Compare versioning, replication and backups

Create a decision table for accidental overwrite, accidental deletion, malicious administrator deletion, AZ outage and Region outage. Versioning helps with retained object versions; replication maintains another copy according to rules; backup retains recovery points; immutable retention restricts deletion. No one mechanism automatically covers every scenario.

For S3 replication, design source/destination buckets with versioning, a narrowly scoped replication role and required encrypted-object permissions. Specify what happens to new objects, pre-existing objects, tags, ownership and delete markers. Existing objects may need a separate batch replication process. Test each rule with one synthetic key before generalizing.

For a cross-account destination, state which account controls destination access and keys. A copy readable only by compromised source credentials does not provide the desired independent recovery boundary. Replication can also propagate unwanted logical changes, so retain an independent recovery strategy.

## 3. Configure a small AWS Backup exercise

If the chosen supported lab resource and Region fit your budget, create a dedicated backup vault without immutable lock. Use a backup plan or an on-demand backup for the dedicated EBS/resource test. Select the explicit resource and service role; do not assign an entire account through a broad tag accidentally.

Record schedule, backup window, retention, lifecycle settings, destination encryption and expected job status. Verify service-specific support for cross-account/Region copies, continuous backups and restore before relying on those options. Features differ by resource and Region.

Restore into a separate isolated lab destination. Keep the source intact. Use the manifest/checksums and application queries to verify data, then check identities, security groups, configuration and integrations. A restored resource may have a new identifier/endpoint; update only the lab application configuration.

If not deploying, use the previously restored EBS volume as evidence and write the exact equivalent plan/vault/resource selections. Label the undeployed Backup workflow clearly.

## 4. Measure a recovery timeline

Record detection, triage, restore initiation, data readiness, application readiness and verification completion. Compare observed RTO with the target. Estimate potential lost changes from the latest valid recovery point, rather than assuming a backup timestamp proves the entire business transaction was captured.

Use the orphan/missing-file test from step one. Report each inconsistency and a reconciliation decision: recreate derived metadata, recover missing original data, or flag the record for human investigation according to ownership. Do not silently mark all rows healthy because counts happen to match.

**Negative test:** remove a hypothetical key permission from the restore role. Explain why a stored backup may become unusable until permission is restored. A locked backup without its decryption dependencies is not complete protection.

## 5. Design immutability without locking a lab

For S3 Object Lock, distinguish governance retention, compliance retention and legal holds. Specify who can bypass governance where permitted, who approves retention length, and how legal holds are reviewed. Enabling Object Lock changes bucket capabilities and cannot be casually reversed.

For AWS Backup Vault Lock, distinguish governance from compliance and its grace-period behavior. Write a pre-lock review containing retention minimum/maximum, recovery-point lifecycle, key dependencies, restore test and ongoing cost estimate. Do not deploy compliance locks for this disposable exercise; retention can become immutable and keep costs unavoidable.

Use a fictional requirement of 30-day retention, then test the effect of an accidental “retain forever” setting. The lesson is to validate lifecycle values before locking, not to perform a destructive experiment.

## 6. Design a hybrid file migration

A fictional office shares 500 GB of files and changes 2 GB daily. Compare DataSync for supported movement/scheduling/verification, Storage Gateway for ongoing hybrid access patterns, and Transfer Family for managed file-transfer protocols and external partners. They serve different workflows.

For DataSync, specify source/destination locations, access identity, connectivity/agent where required, include/exclude rules, metadata expectations, verification and final synchronization. Run a small synthetic subset first: two files, one changed file and one deleted file. Explicitly decide whether deletions propagate.

For Storage Gateway, choose the relevant file/volume/tape pattern, local cache/buffer, disconnected behavior and recovery ownership. Do not assume locally acknowledged behavior and cloud-persisted behavior are identical for every mode. For Transfer Family, define protocol, user mapping, restricted home directory, key/identity source and downstream processing acknowledgment.

Schedule cutover after an initial transfer and final reconciliation. Record source freeze, checksums/counts, permissions, client redirection and rollback trigger. A network transfer completion alone is not business acceptance.

## 7. Add storage visibility

Use S3 Inventory as a scheduled object inventory input where appropriate and Storage Lens for storage-usage patterns. Define questions: which versions grow fastest, which prefixes have no owner, and which incomplete uploads remain? Do not enable every advanced metric without considering scope and cost.

Compare S3 access logs with CloudTrail data events according to the question and coverage. Configure retention and privacy: log records can contain sensitive object identifiers. Keep logs in a protected destination with limited writer/readers, and test delivery rather than assuming enabling an option guarantees continuous evidence.

## Verify, troubleshoot, and clean up

Pass with a successful checksum/application restore, explicit mismatch detection, recovery times and a migration reconciliation table. If jobs fail, inspect resource support, role permissions, KMS/key policy, destination access, service quotas and network dependencies. If restores are slow, distinguish provisioning, data initialization, application warmup and validation.

**Cleanup/cost:** delete dedicated restored resources and unneeded recoverable lab points according to their unlocked lifecycle; remove temporary plans/assignments/locations/agents/logs only when unused. Check destination copies, retained snapshots, transfer endpoints and local appliances. Design-only immutability requires no irreversible cloud changes.

Sources: [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/), [Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html), [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html), [DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/), [Storage Gateway](https://docs.aws.amazon.com/storagegateway/latest/userguide/), [Transfer Family](https://docs.aws.amazon.com/transfer/latest/userguide/).
