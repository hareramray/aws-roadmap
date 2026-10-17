# Measure useful work before buying cheaper capacity

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial combines a reproducible local optimization with a synthetic commitment model. You will verify equivalent output, distinguish utilization from coverage, and evaluate rightsizing, interruptions, capacity assurance, licensing, and sustainability. No reserved capacity or Savings Plan purchase is required.

## 1. Establish a repeatable benchmark

Create `benchmark.py`:

```python
import hashlib
import json
import statistics
import time
from functools import lru_cache

inputs = [20000, 25000, 30000, 35000] * 50

def compute(n):
    return sum(i * i for i in range(n))

@lru_cache(maxsize=16)
def cached(n):
    return compute(n)

def run(function):
    started = time.perf_counter()
    output = [function(n) for n in inputs]
    elapsed = time.perf_counter() - started
    checksum = hashlib.sha256(json.dumps(output).encode()).hexdigest()
    return elapsed, checksum

baseline = [run(compute) for _ in range(3)]
optimized = []
for _ in range(3):
    cached.cache_clear()  # include cold population in every comparison
    optimized.append(run(cached))
assert len({checksum for _, checksum in baseline + optimized}) == 1
for name, results in [("baseline", baseline), ("cached", optimized)]:
    duration = statistics.median(t for t, _ in results)
    print(name, {"median_seconds": duration,
                 "successful_units_per_second": len(inputs) / duration,
                 "checksum": results[0][1]})
print("PASS: equivalent output")
```

Run it on an otherwise quiet machine. Expected output includes equal checksums and a PASS line. The cached version will commonly be faster for this repeated-input workload, but record your measurements rather than promising a universal speedup. The result includes cold cache population and does not claim a production latency improvement.

## 2. Challenge the optimization

Replace the inputs with mostly unique values. Caching now provides little reuse and adds memory/lookup overhead. Reduce cache size below the working set and observe evictions. This demonstrates why a high-level “add caching” recommendation is incomplete without access-pattern evidence.

The function is pure: equal input produces equal output. A real catalog cache also needs freshness and invalidation. A payment operation cannot be cached simply because its HTTP payload repeats. Define what data is authoritative and which stale results are acceptable.

For a real API, measure success rate, throughput, p50/p95/p99 latency, CPU, memory, connections, storage latency, and dependency saturation. Compare the same workload shape and data volume. A faster benchmark that returns fewer records or skips authorization is invalid.

## 3. Optimize in an order that avoids locking in waste

Inventory unused volumes, snapshots, addresses, load balancers, clusters, and development environments. Then schedule appropriate idle periods, rightsize with peak/failure headroom, improve queries/code/data movement, and only then evaluate commitments.

Use Compute Optimizer and Cost Optimization Hub as evidence sources. Confirm their observation period covers relevant demand, inspect memory and non-CPU constraints, and check whether recommendations overlap. Two alternative changes to the same resource cannot necessarily deliver both displayed savings together.

For Graviton or another architecture, verify runtime, native dependencies, container architecture, and software licensing. Benchmark cost per completed useful job with actual rates and execution time. An instance with a lower hourly price can be more expensive per job if it runs much longer or needs more retries.

## 4. Calculate commitment coverage and utilization

Use abstract eligible-usage units per hour, not current AWS prices. Suppose demand is 16 units/hour for 12 hours and 4 units/hour for the remaining 12 hours. Consider a commitment of 8 units/hour.

```python
demand = [16] * 12 + [4] * 12
commitment = 8
eligible_usage = sum(demand)
purchased = commitment * len(demand)
used_commitment = sum(min(value, commitment) for value in demand)
coverage = used_commitment / eligible_usage
utilization = used_commitment / purchased
print({"usage": eligible_usage, "commitment_purchased": purchased,
       "commitment_used": used_commitment,
       "coverage": coverage, "utilization": utilization})
```

Expect usage 240, purchased 192, used 144, coverage 0.60, and utilization 0.75. The commitment covers 60% of eligible demand and uses 75% of what was purchased. Increasing it can improve coverage while worsening utilization during quiet hours.

Repeat with commitments 4, 6, 8, and 12. Build a table of coverage, utilization, uncovered usage, and unused commitment. Then apply actual eligible-service terms and current rates if evaluating a real purchase. A synthetic unit model teaches the shape of the decision but does not reproduce AWS discount allocation rules or contract economics.

## 5. Separate discounts, interruptions, and capacity assurance

On-Demand fits uncertainty and short experiments. Savings Plans and Reserved Instances can discount eligible predictable usage, but scope, term, flexibility, sharing, payment, and capacity implications differ. Read the specific offering instead of assuming every reservation works identically.

Capacity Reservations address supported capacity-assurance needs and can create charges for unused reserved capacity. A discount does not automatically guarantee an instance can launch in the required location. If an incident requires additional capacity, consider quotas, actual availability, and reservation terms in the recovery plan.

Spot fits interruption-tolerant work. Design checkpointing, rescheduling, idempotency, and alternative capacity. Rehearse a worker disappearing after writing a checkpoint but before acknowledging completion. Verify resumed work does not duplicate a business effect and that a partial output cannot be mistaken for a completed result.

Do not buy any commitment to complete this tutorial. Use recommendations and historical data as inputs to a documented decision, then have the organization's actual purchasing process assess it.

## 6. Include licensing and operational cost

Create a software inventory with product, version, owner, license metric, eligible deployment, renewal, and evidence location. AWS License Manager can support relevant license tracking/governance workflows, but it does not replace the license contract or provide blanket permission to move software.

For a Windows or commercial database workload, compare license-included and eligible bring-your-own-license paths only after checking current terms. Dedicated infrastructure may address a contractual requirement but brings its own cost and capacity decisions. Include support, maintenance, migration, and audit work in total cost of ownership.

If a recommendation saves 20 fictional currency units/month but takes weeks to implement and introduces material reliability risk, record that tradeoff. Prioritize expected savings, engineering effort, confidence, reversibility, and business impact rather than sorting only by percentage discount.

## 7. Connect efficiency with sustainability carefully

The cache demonstration reduces repeated CPU work for a suitable workload. Other practical changes include deleting unneeded derived data, compacting small analytical files, reducing network transfer, and scaling down idle capacity. Measure resource utilization and actual workload outcomes.

Do not translate a 30% cost reduction into an unsupported 30% carbon reduction. Prices, hardware, regions, and allocation methods differ. Use supported sustainability data where available and preserve security, residency, availability, and retention requirements.

## Verification, troubleshooting, cost, and cleanup

Deliver benchmark inputs, environment details, three-run results, checksums, unique-input counterexample, commitment table, and one written recommendation with risks. A successful optimization preserves output and meets the chosen performance objective; it need not win every workload.

If timings fluctuate, inspect background load, warm-up, cache state, I/O, thermal limits, and sample size. If a smaller resource fails only at peak, check tail latency and failure headroom rather than average CPU. If estimated savings do not appear, inspect billing delay, discounts, overlapping changes, and the cost view.

The baseline costs no AWS money. For an optional bounded benchmark, record current instance/storage/network/observability charges, cap duration, and stop load generators before deleting the fleet. Remove retained volumes, snapshots, images, public addresses, and logs. Leave no reservation or subscription merely because an instance was terminated.

References: [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/), [Cost Optimization Hub](https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization-hub.html), [Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html), [License Manager](https://docs.aws.amazon.com/license-manager/latest/userguide/license-manager.html), [Sustainability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/sustainability-pillar/welcome.html).
