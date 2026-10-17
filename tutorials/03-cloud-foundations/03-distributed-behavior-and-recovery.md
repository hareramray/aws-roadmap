# Tutorial 03.3 — Demonstrate retries, duplicates, backpressure, and recovery

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../03-cloud-foundations.md)

**Mode:** runnable local PowerShell simulation plus a failure-design exercise. **Result:** evidence explaining why successful cloud APIs alone do not prove reliable application behavior. Reviewed 2026-10-18.

## 1. Simulate an ambiguous result

The document portal charges a fictional processing credit for each accepted job. A server may commit the charge while the response is lost. The client sees a timeout and retries. Run this deliberately simplified local example:

```powershell
$ledger = [System.Collections.Generic.List[string]]::new()
function Add-NaiveCharge([string]$requestId) {
    $ledger.Add($requestId)
}
Add-NaiveCharge 'upload-001'
# Pretend the server response was lost after the write.
Add-NaiveCharge 'upload-001'
$ledger.Count
```

Expected result: `2`. The network failure did not mean the first operation failed. A retry policy attached to the client cannot determine the business outcome without help from the server's data model.

## 2. Introduce an idempotency key

Run a second toy model:

```powershell
$accepted = @{}
function Add-IdempotentCharge([string]$requestId, [string]$fileHash) {
    if ($accepted.ContainsKey($requestId)) {
        if ($accepted[$requestId] -ne $fileHash) {
            throw 'Idempotency key reused with a different request'
        }
        return 'Already accepted'
    }
    $accepted[$requestId] = $fileHash
    return 'Accepted'
}
Add-IdempotentCharge 'upload-001' 'dummy-hash-a'
Add-IdempotentCharge 'upload-001' 'dummy-hash-a'
$accepted.Count
```

Expected responses are `Accepted`, `Already accepted`, and `1`. Then call the function using the same ID with `dummy-hash-b`; it should throw. Reusing an ID for a different business request must not silently return a misleading success.

This is a **single-process teaching simulation**, not production concurrency control. Two processes could both check absence before writing. A real implementation needs a durable conditional insert or transaction and a defined retention period for idempotency records. The operation's business effect and deduplication state must be coordinated; storing the key only after an unrelated payment succeeds leaves another failure window.

## 3. Work through bounded retries

Write a retry schedule with a maximum of four attempts and a total operation deadline of ten seconds. Distinguish retryable throttling/transient errors from invalid input and permission denials. Do not retry an authorization failure forever.

Calculate an exponential base delay, cap it, and add jitter so thousands of clients do not retry in lockstep. For example, choose a random delay between zero and the capped backoff for each attempt. The exact numbers are workload decisions, not AWS universal defaults. Include processing time in the total deadline.

**Negative check:** a downstream call takes six seconds and the caller's deadline is ten. Four more six-second attempts cannot satisfy that deadline. Cancel or stop retrying when the remaining budget cannot support another useful attempt. Repeated retries at every layer can multiply traffic; decide where retry ownership belongs.

## 4. Model a queue and backpressure

Start with a burst of 120 jobs. Two workers each process three jobs per second, so the ideal initial drain time is 20 seconds. If new jobs arrive at eight per second, a six-per-second service rate grows the backlog by two every second. A queue stores work; it does not fix an insufficient processing rate.

Add the oldest-message age to your worksheet. A long queue of tiny jobs can be acceptable, while one stuck high-value job can violate an objective even with a short queue. Track successful completion and age, not just dequeue count.

Introduce a poison message with an invalid image format. After a bounded number of failed attempts, move it to a designated failure path. A dead-letter queue needs an owner, alert, retention and redrive procedure. Fixing code and redriving must preserve the original business ID, so idempotency still works.

Apply backpressure when the database reaches its safe write rate: bound worker concurrency, slow admission if needed, and expose a useful user status. Scaling every worker independently without regard to the database can turn a recoverable bottleneck into an outage.

## 5. Separate authoritative and derived state

For the portal, mark originals and accepted order/history records as authoritative. Mark thumbnails and caches as derived. State what happens if an instance is replaced: local scratch files can disappear; original files remain in durable storage; incomplete work is retried or reconciled.

Version the message schema. Add a new optional field while an old worker still exists, then remove a field the old worker requires. The first can be backward-compatible if handled correctly; the second breaks rolling deployment. Write a compatibility rule and a rollback expectation before changing schema.

Clock skew also matters. Do not resolve financial correctness using arbitrary client timestamps. Use authoritative database conditions or transaction semantics. If a TTL is used for cleanup, separate it from exact authorization expiry checks.

## 6. Find shared failure dependencies

Draw two application instances in separate AZs. Then connect both to one database, one NAT path, one secret, one deployment image and one external identity provider. Circle each dependency that can affect both replicas. Multi-AZ compute alone does not remove these shared failure modes.

Extend the model to tenants or cells: a per-tenant rate limit can stop one customer from exhausting a shared pool. Account boundaries can reduce administrative blast radius. Region boundaries change dependencies but add replication and conflict-handling concerns. Choose each boundary from a measured requirement rather than assuming more boundaries are always better.

## 7. Test RTO/RPO with a recovery timeline

Use a hypothetical last verified backup at 10:00 and a failure at 10:40. Recovery completes at 12:10. The observed recovery duration is 90 minutes; potentially lost changes cover 40 minutes if no additional recovery source exists. Compare those results with the previous tutorial's four-hour RTO and one-hour RPO.

Then change the failure to compromised credentials that also deleted the backup. The earlier test no longer proves recovery. Add independent permissions, protected copies and key availability to the plan. Replication may copy unwanted writes; backup retention needs a separate rationale.

## Verify, troubleshoot, and finish

Keep the console output and a table of expected versus observed behavior. You pass if duplicate submission has one effect, conflicting reuse is rejected, retries respect a deadline, backlog math is correct, and every recovery claim names its evidence.

If the toy functions behave unexpectedly, start a fresh PowerShell session or reset the two local collections. Do not infer distributed safety from a local hashtable. **Cleanup/cost:** the simulation creates only process memory; close the session and retain your report. No AWS charge or cloud deployment is involved.

Sources: [AWS Builders' Library on retries and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/), [fault isolation](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/abstract-and-introduction.html), [reliability pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html).
