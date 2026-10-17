# Validate scientific work, quantum simulation, and specialist migrations

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial uses local numerical work and design calculations to cover HPC, quantum, satellite processing, and legacy ledger migration. It does not run a paid quantum task, reserve satellite capacity, or deploy a blockchain network. The goal is reproducible evidence and a clear service-selection decision.

## 1. Benchmark independent jobs before creating a cluster

Save `parallel_demo.py`. The `__main__` guard is necessary for safe process creation on Windows.

```python
import json
import time
from concurrent.futures import ProcessPoolExecutor
from pathlib import Path

def work(n):
    return sum(i * i for i in range(n))

def main():
    jobs = [150000, 175000, 200000, 225000, 250000, 275000]
    begin = time.perf_counter()
    serial = [work(n) for n in jobs]
    serial_seconds = time.perf_counter() - begin
    begin = time.perf_counter()
    with ProcessPoolExecutor(max_workers=2) as pool:
        parallel = list(pool.map(work, jobs))
    parallel_seconds = time.perf_counter() - begin
    assert serial == parallel
    Path("results.json").write_text(json.dumps(dict(zip(map(str, jobs), parallel)), indent=2))
    print({"jobs": len(jobs), "serial_seconds": serial_seconds,
           "parallel_seconds": parallel_seconds,
           "speedup": serial_seconds / parallel_seconds})
    print("PASS: equal numerical outputs")

if __name__ == "__main__":
    main()
```

Expect equal outputs. The parallel run may be slower for these small tasks because process startup and coordination dominate. Increase work size only within a comfortable local runtime and repeat measurements. More workers are not automatically more efficient.

Treat `results.json` as a simple checkpoint manifest. A resumable version should validate the code/input version, reuse only complete matching results, and recompute missing jobs. Simulate removing one result and show that only that job needs recomputation. For non-pure jobs, stable operation IDs and atomic output publication are required to prevent duplicate business effects.

## 2. Choose HPC execution and storage

Independent jobs fit high-throughput batch; tightly coupled MPI jobs communicate frequently and need different networking/placement decisions. AWS Batch suits supported queued jobs. AWS Parallel Computing Service provides a managed Slurm path. ParallelCluster offers a different cluster-management model. Choose based on scheduler, operating control, applications, and team ownership.

For each option specify job input, container/software version, CPU/memory/GPU needs, maximum runtime, retry/checkpoint policy, output manifest, and cost owner. Use EFA and placement strategies only when the workload and supported instance/network configuration benefit. FSx for Lustre can fit shared high-throughput data access; S3 can hold durable inputs/outputs and staging artifacts. Measure transfer and metadata overhead rather than assuming compute is the only bottleneck.

For accelerators, verify model/software support, memory fit, batch size, and utilization. GPUs, Trainium, Inferentia, and other hardware are not interchangeable binaries. Include licenses and failed jobs in cost per scientific result. Spot requires interruption handling and alternative capacity; a checkpoint that was never tested for restart is not evidence of recoverability.

## 3. Simulate a small quantum measurement

A Bell-state exercise ideally produces outcomes 00 and 11 with equal probability. The following is a classical sampling model of that ideal distribution, not a quantum computer or a full circuit simulator.

```python
import random
from collections import Counter

rng = random.Random(7)
shots = 1000
counts = Counter("00" if rng.random() < 0.5 else "11" for _ in range(shots))
assert set(counts) <= {"00", "11"}
assert sum(counts.values()) == shots
print(dict(counts))
print("PASS: only ideal Bell outcomes sampled")
```

Expect approximately half the shots in each allowed outcome, with finite-sample variation. Add an explicitly simulated bit-flip error to study why noisy outcomes can appear. Do not report the resulting synthetic error rate as measured hardware quality.

To deepen the exercise, use the current Amazon Braket SDK local simulator to construct Hadamard on qubit 0 followed by controlled-NOT to qubit 1, then measure. Verify the circuit representation and compare counts against the ideal model. Learn gates, amplitudes, measurement, shots, noise, and supported device constraints before remote execution.

For a remote Braket extension, choose a currently supported simulator/device, verify queue/availability, cap tasks and shots, estimate task/shot/runtime charges, and check result storage. Hybrid quantum-classical jobs add their own compute and lifecycle. No practical quantum advantage follows merely from successfully running this circuit.

## 4. Design a satellite data pipeline

Use a fictional eight-minute usable contact with 50 Mbps effective downlink. Raw transferred data is `480 × 50,000,000 / 8 = 3 GB` decimal. At an assumed 80% useful-payload fraction, the planning payload is 2.4 GB. Confirm whether “effective” bandwidth already includes overhead before multiplying another efficiency factor in a real estimate.

The design is contact planning → compatible ground/antenna path → authenticated data delivery → durable landing storage → checksum/sequence validation → processing → catalog/analysis. AWS Ground Station provides relevant ground-station capabilities, while satellite compatibility, authorization, orbital scheduling, signal processing, and partner responsibilities need specialist ownership.

Create a manifest for a synthetic downlink: expected packet/file IDs, sequence range, bytes, checksums, contact ID, observation time, and processing version. Remove one file and confirm completeness validation fails. Reprocess an already received contact and ensure catalog records do not duplicate. Distinguish observation time from downlink time when calculating freshness.

Tabletop a missed contact, partial downlink, corrupted payload, delayed processing, and expired delivery permissions. Define whether another contact can recover the data and how long onboard/source retention lasts. This exercise does not require scheduling an actual satellite contact.

## 5. Assess whether a ledger is needed

Write the trust problem before choosing blockchain: who writes, who validates, which parties distrust one another, what finality is needed, how disputes are resolved, and how keys are recovered. A conventional database with constrained writers, signed records, immutable evidence storage, and audit controls may satisfy many requirements with less complexity.

If a distributed ledger is justified, distinguish public/private participation, consensus/finality, transaction fees, contract upgrades, private data handling, and key custody. A lost signing key and a compromised signing key require different responses. Do not put secrets or personal records on an immutable public ledger merely because a hash or API is available.

QLDB has ended support. Managed Blockchain has an announced September 29, 2027 end-of-support date and a new-customer cutoff described in its current notice. Treat both as migration/legacy study branches rather than default new tutorial deployments.

## 6. Produce a lifecycle migration register

Create a table with service/feature, workload owner, current usage, lifecycle state, announced deadline, replacement options, export method, compatibility gaps, validation, rollback, and decommission evidence. Include an existing QLDB-style journal, a Managed Blockchain application, and one retired media/IoT dependency from an old course.

For a journal migration, preserve record identifiers, ordering, timestamps, integrity evidence, and query/reconciliation requirements. Export using the existing service's supported/available path and validate against a trusted manifest. Do not claim a generic database automatically preserves every cryptographic verification or ledger property; explicitly redesign the required trust controls.

For a blockchain application, inventory nodes, data/API dependencies, wallets/keys, contracts, transaction submission, event indexing, and off-chain state. Compare a supported provider/self-managed path or a changed application architecture. Rehearse against a nonproduction network or exported dataset before modifying production signing/transaction paths.

Distinguish maintenance, sunset, and full shutdown. A maintenance product may continue for existing customers while blocking new onboarding. A product page remaining online is not proof that a new account can use it. Feature-level notices should not be generalized to every capability of the parent service.

## Verification, troubleshooting, cost, and cleanup

Deliver equal serial/parallel outputs, measured timings, a checkpoint-reuse plan, ideal quantum counts, the downlink calculation, a detected missing-data case, and a lifecycle migration register with acceptance tests. Label simulated, estimated, and observed results separately.

For poor HPC scaling, inspect task size, communication, data staging, storage contention, memory, and license concurrency. For unexpected quantum results, inspect circuit, measurement basis, shot count, simulator/noise configuration, and device constraints. For satellite gaps, distinguish missing acquisition, transport corruption, and processing failure.

Local work has no AWS charges. Optional AWS costs include scheduler/compute, filesystems, accelerators, licenses, notebooks, quantum tasks/shots, storage, transfer, and specialist service commitments. Stop jobs and producers, then remove dedicated clusters, workers, filesystems, notebooks, task artifacts, and logs. Never schedule a contact, buy capacity, or submit a real ledger transaction simply to complete this tutorial.

References: [AWS PCS](https://aws.amazon.com/pcs/), [AWS Batch](https://docs.aws.amazon.com/batch/), [ParallelCluster](https://docs.aws.amazon.com/parallelcluster/), [Braket](https://docs.aws.amazon.com/braket/latest/developerguide/what-is-braket.html), [Ground Station](https://docs.aws.amazon.com/ground-station/), [Managed Blockchain lifecycle](https://docs.aws.amazon.com/managed-blockchain/latest/hyperledger-fabric-dev/managed-blockchain-end-of-support.html), [AWS shutdown register](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html).
