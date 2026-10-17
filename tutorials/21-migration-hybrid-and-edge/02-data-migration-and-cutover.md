# Reconcile a snapshot and change stream before switching writers

[Domain index](README.md) · [All tutorials](../README.md)

You will simulate snapshot-plus-change capture, detect missing changes, estimate transfer time, and write a cutover/rollback plan. The local model is deliberately small enough to inspect manually. It illustrates correctness requirements that remain when using AWS DMS, DataSync, or another migration tool.

## 1. Define the source contract

Assume a source database maintains current orders and an ordered change position. Each change has a sequence number, operation, key, and new value. The migration starts from a consistent snapshot at sequence 100. Sequence numbers below are synthetic and do not represent a specific database's log format.

Save `cdc_demo.py`:

```python
import hashlib
import json

snapshot = {"A": {"amount": 100, "status": "accepted"},
            "B": {"amount": 200, "status": "accepted"}}
changes = [
    {"seq": 101, "op": "upsert", "id": "A", "value": {"amount": 100, "status": "shipped"}},
    {"seq": 102, "op": "upsert", "id": "C", "value": {"amount": 300, "status": "accepted"}},
    {"seq": 103, "op": "delete", "id": "B"},
    {"seq": 104, "op": "upsert", "id": "C", "value": {"amount": 350, "status": "accepted"}},
]

def apply(snapshot_data, events, start=100):
    state = json.loads(json.dumps(snapshot_data))
    watermark = start
    seen = {}
    for event in sorted(events, key=lambda e: e["seq"]):
        seq = event["seq"]
        encoded = json.dumps(event, sort_keys=True)
        if seq in seen:
            if seen[seq] != encoded:
                raise ValueError("conflicting duplicate change")
            continue
        if seq != watermark + 1:
            raise ValueError(f"missing position before {seq}")
        seen[seq] = encoded
        if event["op"] == "delete":
            state.pop(event["id"], None)
        elif event["op"] == "upsert":
            state[event["id"]] = event["value"]
        else:
            raise ValueError("unknown operation")
        watermark = seq
    return state, watermark

source_final, _ = apply(snapshot, changes)
target, watermark = apply(snapshot, changes + [changes[1]])
assert target == source_final
assert watermark == 104
assert "B" not in target and target["C"]["amount"] == 350
checksum = hashlib.sha256(json.dumps(target, sort_keys=True).encode()).hexdigest()
print({"rows": len(target), "watermark": watermark, "sha256": checksum})
try:
    apply(snapshot, [changes[0], changes[2], changes[3]])
    raise AssertionError("gap should be detected")
except ValueError as error:
    print("EXPECTED FAILURE:", error)
```

Expect two current rows, watermark 104, a deterministic checksum, and an expected missing-position failure. The model sorts a bounded test input and assumes contiguous positions. Real database logs can have transaction and position semantics that differ; use the engine's actual consistency and ordering contract.

## 2. Interpret the result as migration evidence

The delete must remove B, the correction must leave C at 350, and the repeated change must not double an effect. Count-only validation would not detect every incorrect amount. Compare record counts, key sets, checksums or normalized values, constraints, aggregates, and representative application queries.

A source snapshot and CDC start point must align. If they do not, a record can be missed or applied twice. Preserve source positions and transaction boundaries where required, and define handling for deletes, schema changes, large objects, and unsupported types.

Create a schema mapping worksheet for a heterogeneous migration: source type, target type, conversion, null/default behavior, collation/time-zone treatment, precision, and validation. A timestamp or decimal silently changed by conversion can pass a row-count check while breaking the application.

## 3. Map to AWS DMS and schema conversion

For a live extension, verify the selected source/target engines, versions, connection permissions, network paths, and CDC prerequisites. Assess schema conversion separately from data transfer. Stored procedures, extensions, indexes, users, grants, and application SQL may need work outside a basic replication task.

Choose full load, CDC, or full load plus CDC according to the migration. Configure table mappings, error handling, logging, validation, and capacity using current documentation. Begin with a tiny synthetic database and inspect task events and validation results before scaling.

Monitor source change position, replication latency, target errors, storage, and throughput. If the source produces changes faster than they can be processed, the backlog does not converge. A task reporting running is not proof that the target is ready for cutover.

## 4. Estimate file-transfer feasibility

Suppose 500 GB decimal must move over a path delivering 200 Mbps usable throughput. Ideal time is `500 × 10^9 × 8 / (200 × 10^6) = 20,000 seconds`, about 5.56 hours. If measured effective throughput is only 60% of that assumption, time becomes approximately 9.26 hours, before final validation or change reconciliation.

Include source disk speed, destination writes, encryption, small-file metadata, protocol overhead, competing traffic, retries, and the ongoing change rate. Measure a representative sample; one large file does not predict performance for millions of small files.

DataSync suits supported online file/object movement and verification. Choose task scope, source/destination permissions, metadata behavior, scheduling, transfer options, and verification deliberately. Transfer Family addresses managed file exchange with partners; Storage Gateway addresses ongoing hybrid access/caching or volume/tape patterns. These are different jobs rather than interchangeable migration buttons.

For file validation, create a manifest of relative path, size, and supported checksum for a tiny test directory. Copy it to an isolated destination and compare the manifest. Add a changed file and a deleted file to test the selected synchronization semantics. Do not assume every tool deletes destination extras or preserves every metadata field by default.

## 5. Write a timed cutover runbook

Use the following sequence with named owners and explicit go/no-go evidence:

1. Confirm backups, recovery access, target health, and communication readiness.
2. Stop or fence source writers through the documented application procedure.
3. Record the final source transaction/change position and accepted-order count.
4. Wait until the target has applied that position and validation passes.
5. Activate target application configuration and route test traffic.
6. Run login, read/write, permissions, scheduled work, and integration checks.
7. Enable normal traffic and observe the stabilization metrics.
8. Record acceptance and preserve the source according to the retention/rollback plan.

Lower DNS TTL ahead of the window if appropriate, but account for existing connections and client caches. A DNS update does not fence old application writers. Define the write authority independently of routing.

## 6. Rehearse the rollback boundary

Before the target accepts new writes, rollback may be a controlled return to the still-current source after verifying no divergence. After target writes, a simple endpoint reversal can lose or fork data. Define reverse replication, export/reconciliation, or a roll-forward recovery path before that point.

Add one target-only order D to the local model. Show that returning to `snapshot` or an old source copy loses D. Write how D would be reconciled, including conflicts and external effects. Do not pretend that replaying a database row also reverses an already sent shipment or payment.

Decommission only after business acceptance, scheduled-cycle validation, backup verification, and retention obligations. Include credentials, replication agents/tasks, firewall exceptions, staging files, monitoring, licenses, and cost attribution in handover.

## Verification, troubleshooting, cost, and cleanup

Deliver the successful two-row state, watermark 104, detected gap, schema mapping, transfer calculation, and rollback-after-target-write explanation. Label simulated timings and values honestly.

For mismatches, inspect source snapshot alignment, duplicate keys, ordering, missing deletes, type conversion, encoding, time zones, and filter scope. For slow replication, compare source and target bottlenecks rather than merely increasing one task's size. For file mismatch, separate content differences from supported metadata differences.

Local work incurs no AWS charges. Live migration adds temporary double-running, replication capacity, storage, transfer, logging, target databases, and network dependencies. Stop producers/tasks, remove disposable targets and staging data, revoke temporary credentials, and retain only required evidence. Never delete source data as part of a tutorial cleanup before validation is complete.

References: [AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html), [AWS DataSync](https://docs.aws.amazon.com/datasync/), [AWS Transfer Family](https://docs.aws.amazon.com/transfer/), [Storage Gateway](https://docs.aws.amazon.com/storagegateway/).
