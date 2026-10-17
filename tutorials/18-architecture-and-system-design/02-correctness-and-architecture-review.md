# Prove duplicate-safe behavior before reviewing the architecture

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial makes the previous design's correctness claims testable. You will implement an atomic order/outbox write locally, simulate duplicate publication, and review tenant isolation, caching, distributed workflows, and all six Well-Architected pillars. Python and SQLite are sufficient; no cloud deployment is required.

## 1. State business invariants

Use three invariants: an idempotency key identifies one request for one tenant; an accepted order and its outbox record commit together; a duplicate event cannot produce a second local fulfillment record. These are narrower than “exactly once everywhere.” An external provider still needs its own idempotency or reconciliation contract.

Create a new directory and save `outbox_demo.py`:

```python
import hashlib
import json
import sqlite3

db = sqlite3.connect(":memory:")
db.executescript("""
CREATE TABLE orders (
 id INTEGER PRIMARY KEY, tenant TEXT NOT NULL, idem TEXT NOT NULL,
 fingerprint TEXT NOT NULL, amount INTEGER NOT NULL,
 UNIQUE(tenant, idem));
CREATE TABLE outbox (
 id INTEGER PRIMARY KEY, order_id INTEGER NOT NULL,
 published INTEGER NOT NULL DEFAULT 0);
CREATE TABLE fulfillment (
 event_id INTEGER PRIMARY KEY, order_id INTEGER NOT NULL);
""")

def create_order(tenant, key, amount):
    fingerprint = hashlib.sha256(json.dumps({"amount": amount}, sort_keys=True).encode()).hexdigest()
    with db:
        existing = db.execute(
            "SELECT id, fingerprint FROM orders WHERE tenant=? AND idem=?",
            (tenant, key)).fetchone()
        if existing:
            if existing[1] != fingerprint:
                raise ValueError("idempotency key reused with different request")
            return existing[0]
        cursor = db.execute(
            "INSERT INTO orders(tenant,idem,fingerprint,amount) VALUES(?,?,?,?)",
            (tenant, key, fingerprint, amount))
        order_id = cursor.lastrowid
        db.execute("INSERT INTO outbox(order_id) VALUES(?)", (order_id,))
        return order_id

def read_order(tenant, order_id):
    return db.execute("SELECT id,amount FROM orders WHERE tenant=? AND id=?",
                      (tenant, order_id)).fetchone()

def consume(event_id, order_id):
    with db:
        db.execute("INSERT OR IGNORE INTO fulfillment VALUES(?,?)", (event_id, order_id))

first = create_order("tenant-a", "key-1", 2500)
assert create_order("tenant-a", "key-1", 2500) == first
assert read_order("tenant-b", first) is None
try:
    create_order("tenant-a", "key-1", 9999)
    raise AssertionError("changed payload should fail")
except ValueError:
    pass
event_id, order_id = db.execute("SELECT id,order_id FROM outbox").fetchone()
consume(event_id, order_id)
consume(event_id, order_id)  # publish succeeded, acknowledgment was lost
assert db.execute("SELECT count(*) FROM orders").fetchone()[0] == 1
assert db.execute("SELECT count(*) FROM outbox").fetchone()[0] == 1
assert db.execute("SELECT count(*) FROM fulfillment").fetchone()[0] == 1
print("PASS: stable order, rejected changed payload, tenant denial, duplicate-safe local effect")
```

Run `python outbox_demo.py`. Expect the PASS line. This is a single-process teaching model. The unique constraint is essential, but a production concurrent implementation must also handle uniqueness races, isolation, retryable transaction failures, and existing-result retrieval correctly.

## 2. Inspect the crash windows

List four points: before transaction, after order insert but before commit, after commit but before publish, and after publish but before marking published. The transaction protects the first two database operations. The last window can cause duplicate publication. The consumer's unique event ID prevents a second local fulfillment row.

Do not generalize `INSERT OR IGNORE` to every side effect. If the consumer sends an email or charges a card before inserting its deduplication row, a crash can repeat that external effect. If it inserts first and crashes before sending, the effect can be lost. Use an external idempotency key, a durable state machine/outbox, or reconciliation according to the external system's contract.

Add a test case in your notes for a duplicate event with the same ID but a conflicting payload. A robust consumer should detect or reject inconsistency; silently ignoring arbitrary conflicts can hide corruption. Also define how long deduplication history must survive the maximum replay window.

## 3. Choose integration semantics

For one worker pool, a queue buffers work and lets consumers compete. For independent billing and analytics consumers, publish a domain event to separate durable consumer paths. An event bus routes integration events; a retained stream suits ordered partitioned replay and multiple processors. A workflow orchestrator helps make a multi-step business state visible.

Write a contract for `OrderAccepted`: event ID, schema version, tenant, order ID, occurred-at time, producer version, and correlation ID. State ordering scope and duplicate behavior. Keep customer secrets and unnecessary personal fields out of the event.

For an inventory/payment saga, specify compensations: release reserved stock if payment definitively fails; issue a refund if the business accepts that recovery action after a later failure. A compensation is another business action with its own retries and audit trail, not a rollback of time.

## 4. Test authorization and caches

The sample queries by tenant and order ID together. In a real application derive tenant identity from authenticated server-side context, not a request body field. Test tenant A reading tenant B's order, an absent identity, a deleted user, and stale authorization cache entries. A valid login token does not authorize every object.

Compare tenancy models: shared tables with enforced tenant keys, separate schemas/databases, and separate accounts for stronger operational boundaries. Document noisy-neighbor protection, encryption, access reviews, support access, and per-tenant cost attribution. No single model automatically solves all isolation requirements.

For catalog caching, define TTL and invalidation. Simulate a price update while the old value remains cached. Decide whether checkout must revalidate the authoritative price. For order-status caching, define acceptable staleness and ensure cached responses cannot cross tenant boundaries. Add TTL jitter and request coalescing to reduce a cold-cache stampede.

## 5. Plan compatible deployment and graceful degradation

Suppose a new application needs `currency` on orders. First add a compatible nullable/defaulted field, deploy readers that handle both forms, backfill and verify, then enforce the new constraint in a later release. Reverting code does not undo data conversion. Keep roll-forward and restore options distinct.

Define degraded modes: catalog recommendations may be hidden, order intake may remain available while fulfillment queues, and payment ambiguity may create a pending status. A degraded mode must preserve the user contract and show an honest status. Do not return completed merely to keep an availability metric green.

## 6. Review all six pillars using evidence

| Pillar | Evidence from this system | Improvement example |
| --- | --- | --- |
| Operational excellence | Versioned runbook, release ID, failed-event investigation | Add an order reconciliation dashboard |
| Security | Tenant-denial test, scoped roles, sensitive-field policy | Review support access and key lifecycle |
| Reliability | Atomic order/outbox, duplicate handling, restore drill | Test ambiguous external payment response |
| Performance efficiency | Representative latency/load profile | Benchmark query/index changes |
| Cost optimization | Unit-cost model and workload allocation | Remove unnecessary polling or duplicate telemetry |
| Sustainability | Bounded retention and reduced duplicate work | Compact or expire unneeded derived data |

Use the Well-Architected Tool if an AWS workload review is appropriate, recording milestones and improvement ownership. A completed review is not certification of security or availability. Rank improvements by business consequence, probability, effort, and evidence; avoid assigning every issue the same urgent priority.

## Verification, troubleshooting, cost, and cleanup

Acceptance requires the executable tests to pass and a written explanation of the external side-effect gap the sample deliberately does not solve. Review three failures: concurrent duplicate request, unavailable database, and incompatible deployment. For each, identify user behavior, retained state, detection, and recovery.

If duplicates occur, inspect whether the key is stable across retries and scoped to the correct tenant. If legitimate requests conflict, compare request fingerprints and key reuse. If replay changes totals, examine event identity, consumer transaction boundaries, retention, and schema evolution.

The in-memory script has no cloud cost or persistent database to delete. Optional AWS implementation costs include compute, data stores, queues, logs, traces, and temporary release capacity. Remove dedicated stacks, retained tables/snapshots, event subscriptions, queues, publishers, images, and logs after preserving synthetic evidence. Stop producers before deleting consumers to avoid unexplained failed work.

References: [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html), [Saga orchestration pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html), [AWS SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/welcome.html), [Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html).
