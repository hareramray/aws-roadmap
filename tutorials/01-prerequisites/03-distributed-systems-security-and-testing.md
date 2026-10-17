# 03 — Make retries, queues, caches, and failure visible

[Prerequisite tutorials](README.md) · [All tutorials](../README.md)

**Mode:** local simulation and worked design. **Result:** evidence that retries can duplicate work, a durable idempotency pattern, a bounded-retry design, and a security/testing checklist derived from the behavior you observed.

## 1. Reproduce the ambiguous timeout

Imagine a client creates an order. The server commits it, but the response is lost. The client cannot infer from a timeout whether the write happened. A retry with a new order ID can create a second order.

Save as `idempotency_lab.py` and run it locally:

```python
import json
import sqlite3


def submit(connection, request_id, payload):
    serialized = json.dumps(payload, sort_keys=True, separators=(",", ":"))
    with connection:
        row = connection.execute(
            "SELECT payload, result FROM requests WHERE id = ?", (request_id,)
        ).fetchone()
        if row:
            if row[0] != serialized:
                raise ValueError("Idempotency key reused with different payload")
            return json.loads(row[1])
        result = {"order_id": "order-" + request_id, "state": "accepted"}
        connection.execute("INSERT INTO requests VALUES (?, ?, ?)",
                           (request_id, serialized, json.dumps(result)))
        return result


def main():
    with sqlite3.connect(":memory:") as connection:
        connection.execute("CREATE TABLE requests (id TEXT PRIMARY KEY, payload TEXT, result TEXT)")
        first = submit(connection, "r1", {"sku": "BOOK", "quantity": 1})
        # Simulate losing the response; the client repeats the same logical request.
        second = submit(connection, "r1", {"quantity": 1, "sku": "BOOK"})
        assert first == second
        assert connection.execute("SELECT COUNT(*) FROM requests").fetchone()[0] == 1
        try:
            submit(connection, "r1", {"sku": "BOOK", "quantity": 2})
        except ValueError:
            print("PASS: retry reused result and mismatched payload was rejected")
        else:
            raise AssertionError("Payload mismatch was accepted")


if __name__ == "__main__":
    main()
```

This single-process example demonstrates the contract. A distributed implementation must also handle concurrent first requests, crash recovery, in-progress states, transaction boundaries, and retention. A primary key/conditional transaction helps arbitrate concurrency; a Python dictionary alone is insufficient across processes or replacement.

## 2. Extend the simulation to a queue

Represent a queue as a list of request IDs including duplicates. Process each with `submit`. Verify that repeated logical requests do not multiply durable results. Then deliberately use a fresh key per receive and observe the duplicate effect. The stable key identifies the business operation, not the network attempt or receipt handle.

Now imagine a side effect outside the database transaction, such as sending a notification. Record an outbox entry in the same transaction as the order, publish it asynchronously, and design the consumer to tolerate repeated delivery. A transaction in one database does not atomically cover every external service.

## 3. Budget timeouts and retries

Define an overall request deadline, then allocate time among connection setup, attempts, processing, and response handling. If the outer API times out after two seconds but a dependency retries for thirty seconds, work can continue after the caller gives up.

Use bounded retries for retriable errors, exponential delay with jitter, and a maximum elapsed time. Reject permanent validation/authentication errors promptly. Compare three layers each trying three times: worst-case amplification can be 27 downstream attempts if retries nest. Record where the single retry budget should be enforced.

A circuit breaker temporarily stops calls to a failing dependency after a defined condition; it needs recovery probing and monitoring. Load shedding rejects work beyond safe capacity. Backpressure slows producers or limits admitted work. Queueing buffers bursts but can hide overload if the backlog grows without bound.

## 4. Design a cache experiment

Use a local dictionary with `value`, `source_version`, and expiry time. Read through it into a source dictionary. Update the source and observe a stale cached value until invalidation/expiry. Then issue several simultaneous misses and reason about a cache stampede.

Add a per-key single-flight mechanism or carefully bounded refresh, TTL jitter, and a stale-data policy appropriate to the application. Include tenant/authorization and relevant version information in cache keys. Do not cache another user's authorized result under a globally shared question string.

Explain why cache persistence, replication, and backup are separate service features. Do not assume all managed caches are ephemeral or all persistent caches provide the same durability contract.

## 5. Work through consistency and availability

Draw a primary and a lagging replica. A user writes to the primary and immediately reads the replica. Define whether stale results are acceptable and how a read-your-writes requirement could be met. Under a network partition, identify which operations stop or which consistency guarantee is weakened according to the chosen system.

Discuss CAP in the presence of a partition rather than as an unrestricted “choose any two” slogan. Distinguish durability (data survives supported failures), availability (requests can be served), and consistency (what values operations can observe).

Compute a simple capacity estimate: at 100 requests/second with average time in the system of 0.2 seconds, approximately 20 concurrent in-flight requests are implied under steady-state assumptions. This does not set a safe pool size by itself; tail latency, bursts, retries, and resource constraints matter. Measure p95/p99 separately from averages.

## 6. Apply security and tests to the same system

Authenticate the caller, then authorize the order operation and tenant separately. Validate input lengths/types/ranges before storing or invoking a dependency. Use encryption in transit and appropriate storage protection, but recognize that encryption does not correct a permission bug. Store credentials through supported role/secret mechanisms, not code.

Choose tests by failure: unit tests for validation/calculation, transaction tests for rollback, contract tests for API shape, integration tests for permissions/real service semantics, bounded load tests for saturation, and restore tests for recovery. A mocked SDK cannot prove the deployed IAM policy or network path.

Add a request ID and structured outcome logs without secrets. A useful error identifies the failed operation and relevant context without leaking private data. Preserve the root cause when handling exceptions rather than returning a generic success.

## Completion and cleanup

Deliver the duplicate-request result, mismatch rejection, retry-amplification calculation, cache-staleness example, consistency diagram, and a risk-based test plan. The exercise requires no AWS resources. Keep synthetic scripts as evidence or remove their dedicated local directory after reviewing its contents.

Reference: [Amazon Builders' Library](https://aws.amazon.com/builders-library/) for detailed primary-source discussions of timeouts, retries, overload, deployment, and distributed-system operations.
