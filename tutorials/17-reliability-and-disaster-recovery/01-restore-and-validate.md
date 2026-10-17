# Restore an order database and measure what was recovered

[Domain index](README.md) · [All tutorials](../README.md)

This baseline uses Python's built-in SQLite support. It deliberately keeps the original database intact and restores to a different filename. You will verify a usable application record, not just a successful copy command. The later AWS Backup section translates the same evidence into a managed-service workflow.

## 1. Write the recovery contract

For this fictional order system, accepted orders contain an ID, integer amount in minor currency units, and status. The application must never silently change an accepted amount. The exercise recovery target is a ten-minute RTO and a five-minute RPO. These are chosen requirements, not measurements or AWS guarantees.

The failure scenario is logical corruption discovered after a backup. A replicated copy could contain the same corruption, so historical recovery points matter. Preserve a known-good copy, a deterministic manifest, deployment/configuration instructions, and the credentials/key dependencies needed to reopen the store.

Make a new local directory and save `restore_lab.py`:

```python
import hashlib
import json
import sqlite3
import time
from pathlib import Path

names = ["source.db", "backup.db", "restored.db", "manifest.json"]
if any(Path(name).exists() for name in names):
    raise SystemExit("Use an empty directory; existing evidence will not be overwritten")

def rows(connection):
    return connection.execute(
        "SELECT order_id, amount_minor, status FROM orders ORDER BY order_id"
    ).fetchall()

def digest(values):
    raw = json.dumps(values, separators=(",", ":")).encode()
    return hashlib.sha256(raw).hexdigest()

source = sqlite3.connect("source.db")
source.execute("CREATE TABLE orders(order_id INTEGER PRIMARY KEY, "
               "amount_minor INTEGER NOT NULL CHECK(amount_minor >= 0), "
               "status TEXT NOT NULL)")
source.executemany("INSERT INTO orders VALUES (?, ?, ?)",
                   [(i, 1000 + i, "accepted") for i in range(1, 101)])
source.commit()
manifest = {"count": 100, "last_id": 100, "sha256": digest(rows(source))}
backup = sqlite3.connect("backup.db")
source.backup(backup)
backup.close()
Path("manifest.json").write_text(json.dumps(manifest, indent=2))

# These five accepted orders are intentionally newer than the backup.
source.executemany("INSERT INTO orders VALUES (?, ?, ?)",
                   [(i, 1000 + i, "accepted") for i in range(101, 106)])
source.commit()
latest_source_id = source.execute("SELECT max(order_id) FROM orders").fetchone()[0]

start = time.perf_counter()
backup = sqlite3.connect("file:backup.db?mode=ro", uri=True)
restored = sqlite3.connect("restored.db")
backup.backup(restored)
assert restored.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
values = rows(restored)
assert len(values) == manifest["count"]
assert digest(values) == manifest["sha256"]
assert values[-1][0] == manifest["last_id"]
assert restored.execute("SELECT amount_minor FROM orders WHERE order_id=42").fetchone()[0] == 1042
elapsed = time.perf_counter() - start
print(json.dumps({"restored_count": len(values), "restored_last_id": values[-1][0],
                  "newer_source_orders": latest_source_id - values[-1][0],
                  "restore_and_validate_seconds": round(elapsed, 6)}, indent=2))
source.close()
backup.close()
restored.close()
```

Run it once in the empty directory. Expected output includes `restored_count: 100`, `restored_last_id: 100`, and `newer_source_orders: 5`. Elapsed time varies. The checksum proves equality to the chosen manifest, while the query for order 42 demonstrates one application-level expectation.

## 2. Interpret recovery correctly

Five source orders are newer than the recovery point. This is a record-loss observation, not automatically a five-minute RPO. To calculate time-based RPO, retain commit timestamps and compare the last recoverable committed record with the failure time. If orders 101–105 occurred over three minutes, this scenario loses up to those three minutes of accepted work.

The script measures restore plus validation only. A real RTO also includes detection, incident declaration, authorization, provisioning, key access, routing, application startup, and business acceptance. Record those separately rather than reporting a millisecond SQLite copy as the system's DR time.

A full backup check includes structural integrity, expected counts, business invariants, representative queries, and external dependency compatibility. Row count alone would miss an incorrect payment amount. A whole-file hash may vary after a valid database rewrite; a deterministic logical-record hash tests a different property and is appropriate for this exercise.

## 3. Rehearse a failed validation

Copy only `restored.db` to a disposable validation target and change the amount on order 42 using SQLite. Recalculate the logical digest. It must differ from the manifest even though the row count remains 100. Do not overwrite the backup or source to run this test.

Then simulate a missing key/permission by describing the expected AWS equivalent: restore creation may succeed under one role while the application cannot decrypt or read the restored resource. Your runbook must verify both recovery permissions and runtime permissions. A backup administrator and an application role are different identities.

Write a recovery instruction that another learner can follow: prerequisites, exact recovery-point identifier, target location, owner, required permissions, data checks, expected newer records, and acceptance criteria. Include a “stop and investigate” branch if checksums or invariants fail.

## 4. Translate the exercise to AWS Backup

Choose one supported disposable AWS resource, such as a tiny supported database or volume containing only synthetic data. Before deployment, check AWS Backup's current resource/Region feature matrix. Continuous recovery, cross-account copies, cross-Region copies, encryption, and restore testing differ by resource.

Create a dedicated backup vault and a narrowly scoped backup role using the current service guide. Select only resources carrying the tutorial identifier. Configure a short supported retention period and a schedule that will not run indefinitely. Take a recovery point and wait for the actual job to complete. Record resource ARN, backup job ID, recovery point ARN, key, creation time, and intended recovery time.

Restore into a new isolated destination. Some services need subnet groups, security groups, instance classes, parameters, or application reconnection settings supplied at restore time. Read the restore metadata instead of assuming the backup recreated every operational dependency. Execute the same logical record checks and application query used locally.

For automated AWS Backup restore testing, distinguish successful restore orchestration from your own validation. Add application validation where supported and record its result. Inspect cleanup behavior and any resources that automation cannot remove because of permissions or dependencies.

## 5. Protect recoverability against compromise

Draw three administrative roles: workload operator, backup operator, and recovery approver. Explain which role can create, copy, restore, delete, or alter retention. Consider a copy in another account when workload-credential compromise is in scope, and verify that the necessary encryption/key path remains usable there.

Study Vault Lock governance and compliance modes as a design exercise first. Compliance retention becomes intentionally irreversible after its configured grace behavior and can retain billable data. Logically air-gapped vaults use a service-specific protection and sharing model; they are not a promise that every backup is an offline tape. Validate supported resources and recovery access before adopting them.

Set retention from both RPO and corruption-discovery time. A daily backup retained for one day cannot recover a defect found a week later. Likewise, retaining every backup forever without a requirement creates cost and deletion obligations.

## Verification, troubleshooting, and cleanup

Accept the tutorial only when you can restore 100 known records, detect the altered amount, explain the five newer source records, and distinguish restore duration from end-to-end RTO. For a live extension, add an actual recovery-point ID and a successful application query against the new destination.

Common failures include an inconsistent raw file copy, missing transaction flush, incorrect recovery point, wrong key permissions, wrong subnet access, unrecorded application secrets, and validation run against the source by mistake. Print the destination identifier before every check.

Local cleanup is limited to this exercise directory after evidence is saved. AWS cleanup includes restored resources, scheduled selections/plans, disposable recovery points, temporary roles, snapshots, logs, and dedicated networking. Do not delete shared keys or protected production backups. Price storage, restore capacity, copy transfer, encryption requests, and retained recovery points before the optional extension.

References: [AWS Backup feature availability](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html), [restore testing](https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html), [Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html), [SQLite backup API](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.backup).
