# Replay events, handle late data, and build a warehouse projection

[Data tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../14-data-engineering-and-analytics.md)

**Outcome:** Process duplicate and late events deterministically, prevent stale CDC updates, and reconcile a dimensional aggregate. The first exercise is local Python/SQLite. The optional Kinesis experiment uses an existing dedicated stream and incurs usage; Redshift, MSK, Flink, Firehose, OpenSearch, and BI extensions require separate cost and access review before provisioning.

## 1. Separate event time, arrival order, and source version

A stream record has an event ID, entity key, source version, occurrence time, and payload. Arrival order is not necessarily business order. A retry can repeat a record; a late update can have an older source version; a delete must remain effective even if an older update arrives afterward. Retention/checkpoints provide replay, while business keys/version rules determine whether replay changes results.

Save `stream_lab.py`. Event-time units are synthetic integer minutes. The watermark is maximum observed event time minus five minutes, solely to make this lesson deterministic. Events older than it go to a late side output; production watermark policy must reflect actual source characteristics.

```python
import json
import sqlite3
from collections import defaultdict
from pathlib import Path

events = [
    {"id": "e1", "key": "o1", "version": 1, "time": 2, "region": "north", "amount": 100},
    {"id": "e2", "key": "o2", "version": 1, "time": 4, "region": "south", "amount": 200},
    {"id": "e1", "key": "o1", "version": 1, "time": 2, "region": "north", "amount": 100},
    {"id": "e3", "key": "o1", "version": 2, "time": 12, "region": "north", "amount": 150},
    {"id": "e4", "key": "o3", "version": 1, "time": 3, "region": "north", "amount": 50},
    {"id": "e5", "key": "o1", "version": 1, "time": 13, "region": "north", "amount": 100},
    {"id": "e6", "key": "o2", "version": 2, "time": 14, "region": "south", "amount": 0, "deleted": True},
]

def project(records):
    seen, current, late = set(), {}, []
    windows = defaultdict(int)
    maximum_time, duplicates, stale = -1, 0, 0
    for event in records:
        if event["id"] in seen:
            duplicates += 1
            continue
        seen.add(event["id"])
        maximum_time = max(maximum_time, event["time"])
        if event["time"] < maximum_time - 5:
            late.append(event)
            continue
        old = current.get(event["key"])
        if old and event["version"] <= old["version"]:
            stale += 1
            continue
        current[event["key"]] = event  # Keep delete tombstones with their version.
        windows[event["time"] // 10] += 1  # Accepted changes, not revenue.
    return current, late, dict(windows), duplicates, stale

state, late, windows, duplicates, stale = project(events)
assert (duplicates, stale, len(late)) == (1, 1, 1)
assert windows == {0: 2, 1: 2}
assert state["o1"]["amount"] == 150 and state["o2"]["deleted"]
# Full replay rebuilds the same projection; replay twice also preserves its result.
replayed, _, _, _, _ = project(events + events)
assert replayed == state

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE dim_region (region_key INTEGER PRIMARY KEY, region TEXT UNIQUE)")
db.executemany("INSERT INTO dim_region VALUES (?, ?)", [(1, "north"), (2, "south")])
db.execute("CREATE TABLE fact_order (order_id TEXT PRIMARY KEY, region_key INTEGER, amount_cents INTEGER)")
for row in state.values():
    if not row.get("deleted", False):
        region_key = db.execute("SELECT region_key FROM dim_region WHERE region=?", (row["region"],)).fetchone()[0]
        db.execute("INSERT INTO fact_order VALUES (?, ?, ?)", (row["key"], region_key, row["amount"]))
result = db.execute("SELECT d.region, SUM(f.amount_cents) FROM fact_order f JOIN dim_region d ON f.region_key=d.region_key GROUP BY d.region").fetchall()
assert result == [("north", 150)]
Path("events.json").write_text(json.dumps(events, indent=2), encoding="utf-8")
Path("late-events.json").write_text(json.dumps(late, indent=2), encoding="utf-8")
print({"duplicates": duplicates, "stale": stale, "late": len(late), "accepted_change_windows": windows, "warehouse": result})
db.close()
```

Run it and expect one duplicate, one stale update, one late event, windows `{0:2,1:2}`, and north revenue 150. `o3` is absent because the policy quarantined its late arrival; a reconciliation job can later incorporate it explicitly. Do not call this revenue complete without that decision. The window counts accepted changes, whereas the warehouse represents current nondeleted orders; those are different metrics.

## 2. Break the guarantees locally

Remove the source-version check and observe the stale update overwrite `o1` from 150 to 100. Remove tombstones and consider what a later version-1 update would do to deleted `o2`. Remove event-ID deduplication and observe how accepted-change counts can inflate if version rules are also inadequate. Restore the checks after each experiment.

The example's deduplication set and state live in memory. Restarting an incremental consumer without restoring both can replay side effects. A production implementation must checkpoint input position and state consistently and make sink writes idempotent/transactional. A bounded deduplication retention must match the supported replay horizon; unbounded state will eventually exhaust memory.

## 3. Optional: publish a bounded Kinesis sample

Use a dedicated empty, single-shard Kinesis Data Stream in the intended account/Region for this deterministic fixture. Choose provisioned single-shard capacity and retention after estimating cost, record its ARN, and wait for ACTIVE. Require the stream's `USER_PARTITION_KEY` distribution behavior; the current `AUTO` distribution mode ignores supplied partition keys and does not preserve this lesson's key-based assumption. The producer needs `PutRecord` on that stream; a separate reader needs list/describe/get-iterator/get-records permissions. Keep the sample tiny and stop it explicitly. Do not reshard during this exercise.

Save `publish.py`. It previews by default and submits exactly seven records only with `--submit`. The existing `events.json` is the fixture produced above.

```python
import argparse
import json
import os
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("--submit", action="store_true")
args = parser.parse_args()
events = json.loads(Path("events.json").read_text(encoding="utf-8"))
if not args.submit:
    print({"records": len(events), "partition_key": "roadmap-fixture",
           "entity_keys": sorted({e["key"] for e in events})})
    raise SystemExit(0)
import boto3
expected = os.environ["EXPECTED_AWS_ACCOUNT"]
if len(expected) != 12 or not expected.isdigit():
    raise ValueError("Set the expected sandbox account")
session = boto3.Session(region_name=os.environ.get("AWS_REGION") or os.environ.get("AWS_DEFAULT_REGION"))
if not session.region_name:
    raise ValueError("Set an explicit sandbox Region or a profile with a Region")
if session.client("sts").get_caller_identity()["Account"] != expected:
    raise RuntimeError("Wrong account")
client = session.client("kinesis")
receipts = []
try:
    for event in events:
        request = {"StreamName": os.environ["LAB_STREAM"],
                   "PartitionKey": "roadmap-fixture", "Data": json.dumps(event).encode()}
        if receipts:
            request["SequenceNumberForOrdering"] = receipts[-1]["sequence"]
        result = client.put_record(**request)
        receipts.append({"event_id": event["id"], "shard": result["ShardId"],
                         "sequence": result["SequenceNumber"]})
finally:
    client.close()
    Path("publish-receipts.json").write_text(json.dumps(receipts, indent=2), encoding="utf-8")
print({"submitted": len(receipts)})
```

Run the preview first. This diagnostic producer deliberately sends every business entity under one fixture partition key and chains `SequenceNumberForOrdering` to preserve the example's total order. SDK retries or ambiguous acknowledgments can still produce duplicate delivery; the event ID stays stable. In a scaled application, per-entity partition keys provide separate ordering boundaries, and a hot entity can overload one shard. Sequence numbers are stream positions, not a universal business version.

For the read exercise, use `list-shards` and verify exactly one active shard, obtain its `TRIM_HORIZON` iterator, and call `get-records` with a small limit until all expected records have been seen or a 60-second deadline is reached. Advance using `NextShardIterator`; respect current per-shard polling limits and record lag. Decode data as UTF-8 JSON and retain shard sequence order before passing events to the local projection. Check the received event-ID sequence against the published fixture; retries can add duplicates, so record any discrepancy rather than forcing a predetermined answer. Multiple shards do not have a global arrival order: concatenating their batches can change this toy watermark's late/window results. A real multi-partition processor needs an explicit watermark/idle-partition policy. A production consumer must also handle pagination, shard splits/merges, iterator expiry, checkpointing, and multiple workers; use the supported Kinesis Client Library where appropriate. Do not generalize this bounded diagnostic reader into a complete consumer.

## 4. Choose delivery and computation separately

Repeat the design on Kafka/MSK by mapping partition keys, retained offsets, consumer groups, acknowledgment, and replay explicitly. A committed offset alone does not prove an external database write occurred once. For CDC, obtain a consistent initial snapshot plus a known source position, preserve key/version/delete semantics, and reconcile counts with the source before cutover.

For Firehose, configure an isolated destination prefix, bounded buffering, error/backup prefixes, and a transformation only if needed. Submit the same tiny fixture and measure ingestion-to-object delay, then inspect duplicates and rejected transformation records. Firehose buffers delivery; it is not a replacement for arbitrary retained consumer offsets.

For managed Flink, implement the same watermark, window, keyed version state, and late side output using a supported Flink release. Take a checkpoint/savepoint, stop and restore the application, and verify the sink's result after replay. State exactly-once processing depends on source/sink participation and does not automatically cover external APIs. Test state/schema compatibility before upgrades. The discontinued Kinesis Data Analytics SQL service is not a new-build prerequisite.

## 5. Adapt the projection to Redshift and downstream consumers

In a dedicated Redshift schema, create a region dimension and order fact at the same grain as the local example. Load a small validated CSV/Parquet staging dataset from a scoped S3 prefix using an attached role, then reconcile stage counts and sums before merging into the fact table. Deduplicate/resolve source versions in staging so `MERGE` receives one intended source row per target key; apply deletes deliberately. Redshift primary/foreign key constraints are not a substitute for ETL uniqueness validation.

Compare this warehouse query with the local result:

```sql
SELECT d.region, SUM(f.amount_cents) AS revenue_cents
FROM tutorial.fact_order f
JOIN tutorial.dim_region d ON f.region_key = d.region_key
GROUP BY d.region
ORDER BY d.region;
```

Inspect plans, distribution/skew, sort pruning, encodings, statistics, concurrency, and workload controls using a representative dataset. Evaluate materialized views for repeatable aggregates and verify refresh behavior. Serverless versus provisioned selection depends on usage shape and controls; neither removes governance or query costs. Do not create a large warehouse merely to reproduce the in-memory result.

Feed OpenSearch with a purpose-built searchable projection and a deterministic document ID; test upserts/deletes and index retention independently from warehouse retention. In Amazon Quick/QuickSight capabilities or another BI client, define the same metric, refresh/freshness target, and row-level sharing policy. A dashboard screenshot should reconcile to the fact query. Unified Studio can organize projects/catalog access, but it does not remove S3, engine, or dataset ownership duties.

Orchestrate a bounded batch as `ingest -> validate -> transform -> reconcile -> publish -> notify`, with failures stopping publication and retries retaining the same batch ID. Step Functions/Glue workflows suit different integration needs; MWAA/Airflow introduces scheduler/environment lifecycle and cost. Keep every stage's input/output manifest and lineage so a correction can be replayed through dependent aggregates.

## 6. Cost, teardown, and mastery

Stop producers and consumers, inspect lag/late/error outputs, then delete only the dedicated stream after saving required synthetic evidence. Stop/delete Flink applications, MSK clusters, Firehose streams, warehouse namespaces/workgroups or provisioned clusters, OpenSearch domains/collections, and BI datasets created for extensions. Remove associated storage, logs, roles, snapshots, endpoints, and scheduled orchestration. Stream retention and empty persistent infrastructure can continue billing after application code stops.

Explain why replay requires both offsets and sink semantics, why a late record differs from a stale source version, why tombstones matter, and why an accepted-change window is not a revenue metric.

Official references: [Kinesis consumer development](https://docs.aws.amazon.com/streams/latest/dev/developing-consumers-with-sdk.html), [Kinesis PutRecord API](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_PutRecord.html), [Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html), [Managed Flink concepts](https://docs.aws.amazon.com/managed-flink/latest/java/how-it-works.html), [Redshift MERGE](https://docs.aws.amazon.com/redshift/latest/dg/r_MERGE.html), [Amazon Quick overview](https://docs.aws.amazon.com/quicksuite/latest/userguide/what-is.html).
