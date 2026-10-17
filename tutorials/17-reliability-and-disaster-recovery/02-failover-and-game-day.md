# Rehearse recovery without assuming a second Region solves it

[Domain index](README.md) · [All tutorials](../README.md)

This worked design uses explicit traffic and recovery assumptions. The baseline is a tabletop and local calculation, so it creates no AWS resources. An optional game day should use only resources specifically authorized for a controlled experiment.

## 1. Check surviving capacity

Assume a checkout service receives 600 requests/second at peak. A tested application instance handles 120 requests/second while meeting its latency target. Two Availability Zones each have three instances. Normal fleet capacity is 720 requests/second, leaving 120 requests/second of headroom.

Lose one AZ: three surviving instances provide only 360 requests/second. The system is distributed across AZs but cannot maintain peak service. To survive one AZ without waiting for scaling, each AZ needs at least five tested instances, assuming the database and other dependencies also have sufficient capacity. This is arithmetic under stated assumptions, not an instruction to overprovision every workload.

```python
import math
peak = 600
per_instance = 120
zones = 2
instances_per_zone = 3
normal = zones * instances_per_zone * per_instance
surviving = (zones - 1) * instances_per_zone * per_instance
required_per_zone = math.ceil(peak / ((zones - 1) * per_instance))
print({"normal_capacity": normal, "surviving_capacity": surviving,
       "required_instances_per_zone": required_per_zone})
```

Expected results are 720, 360, and 5. Extend the worksheet with connection limits, address space, database write capacity, queue drain rate, and external payment limits. An extra instance cannot compensate for a dependency already saturated at 400 requests/second.

## 2. Make partial failure safe

Define a bounded request deadline, dependency timeout, and retry owner. For the example, use a fictional two-second customer deadline and at most one retry for an idempotent dependency operation. Check that backoff and processing fit the remaining deadline; do not independently give every layer two seconds.

Document ambiguous results. If a payment call times out after the provider accepted it, retry with the same business idempotency key and reconcile the provider's outcome. A new random key on each retry defeats deduplication. If inventory is unavailable, queue an explicitly pending order or decline it according to the product contract; do not silently mark it complete.

Add backpressure: cap admitted work, expose an appropriate retryable response, and track queue age. Define a cache-warming plan because recovery to a cold cache can overload an otherwise correctly sized database. Separate optional recommendations from mandatory payment processing so one dependency does not remove the whole service.

## 3. Select a regional recovery strategy

Use a fictional RTO of 45 minutes and RPO of five minutes. Compare:

| Pattern | Measured/estimated work to validate | Decision for this exercise |
| --- | --- | --- |
| Backup/restore | Infrastructure creation, restore, validation, routing | Candidate only if repeated drills fit 45 minutes |
| Pilot light | Replicated core state plus launching/scaling application | Candidate if provisioning dependencies are reliable enough |
| Warm standby | Functional reduced environment plus scaling | Chosen for the tabletop; costs more before an incident |
| Active/active | Concurrent traffic and data authority across locations | Rejected because current business need does not justify complexity |

Build an explicit timeline: detect 3 minutes, decide 4, promote or recover data 8, scale 10, validate 8, shift traffic 5. Total is 38 minutes, leaving seven minutes of margin. These are estimates until a drill measures them. Parallel steps should be represented as dependencies; do not subtract their duration simply because you hope they overlap.

## 4. Write a data authority matrix

| Store | Normal authority | During failover | Essential verification |
| --- | --- | --- | --- |
| RDS/Aurora writer | Primary writer endpoint | Promote or recover according to the chosen engine/topology | Latest accepted order, reconnect behavior, remaining lag |
| Aurora Global Database | Configured primary writer | Use supported switchover/failover procedure | Planned versus unplanned behavior and possible data loss |
| DynamoDB global tables | Configured MREC or MRSC design | Follow that mode's write/consistency restrictions | Conflicts, supported Regions, transaction/feature limitations |
| S3 objects | Versioned source plus defined replica | Switch only after required objects/versions are usable | Replication status, permissions, encryption, delete behavior |
| Queue and cache | Defined queue authority; cache derived | Replay or rebuild using documented rules | Duplicate-safe processing and controlled cold-start load |

Do not claim every global table is eventually consistent: DynamoDB has distinct MREC and MRSC options. Conversely, choosing a strong-consistency option does not remove every cross-system transaction problem. An order database and an external payment provider remain separate systems.

Fence writes to the old primary before accepting writes at a new single-writer destination where the design requires it. A network partition can leave both locations alive; health checks alone do not establish which one has authority. State who can declare failover and how an operator confirms that split brain has been prevented.

## 5. Pre-position the recovery dependencies

List artifacts, images, infrastructure templates, IAM roles, emergency access, keys, secrets, certificates, DNS records, quotas, monitoring, and vendor allowlists. Mark each dependency's account and Region. Resolve any dependency that still requires the unavailable primary location.

Compare Route 53 routing with Global Accelerator for your protocol and traffic path. DNS caches and existing connections mean clients do not all switch instantly. Amazon Application Recovery Controller can support relevant recovery/routing capabilities; verify the exact feature because ARC Readiness Check has a separate availability notice. Do not make a new exercise depend on that maintenance feature.

For server-based recovery, evaluate AWS Elastic Disaster Recovery's supported replication and drill workflow. Record staging resources, launch settings, application validation, and failback procedures. A recovered VM booting is not proof that its database or business integration is consistent.

Use Resilience Hub, where suitable, to model an application and assess recommendations against recovery objectives. An assessment complements drills; it does not replace measured recovery or establish a contractual guarantee. Compare recommendations with the actual dependency map.

## 6. Conduct a bounded game day

Start with a tabletop: the facilitator declares one AZ unavailable, the operator selects the recovery action, and the note taker records every missing prerequisite. Next simulate an expired certificate or unavailable identity provider; these expose dependencies that a server-kill exercise misses.

An optional AWS Fault Injection Service experiment needs a specific supported action, dedicated target resources, experiment role, duration, steady-state metric, stop alarm, and rollback owner. Begin with one disposable target. Check target selection before starting; resource tags are not safe if they match unrelated infrastructure.

Define stop conditions such as synthetic checkout failures exceeding the lab's allowed window or loss of the administration path. A stop alarm must itself have working data and permissions. Explain which effects stopping an experiment reverses and which require explicit repair.

Rehearse failback separately. Reconcile writes, re-establish replication, verify the intended writer, warm capacity, and gradually move traffic. Changing DNS back without reconciling data can create a second outage.

## Verification, troubleshooting, cost, and cleanup

Deliver the capacity calculation, timeline, data authority matrix, dependency inventory, and a completed drill log. Every time must be labeled estimated or measured. Record at least one discovered gap and retest it after correction.

If failover is slow, break down detection, decision, data, capacity, validation, and routing time. If replicas disagree, stop writes according to the established procedure and reconcile rather than forcing traffic toward whichever copy looks newer. If autoscaling stalls, inspect quotas, capacity, images, subnet space, and bootstrap dependencies.

The baseline costs nothing in AWS. Optional live work can incur standby compute, databases, replication, transfer, FIS, recovery services, logs, and DNS/edge charges. Stop experiments, restore the intended configuration, disable temporary schedules, and remove only dedicated recovery targets, staging resources, and test copies. Keep production recovery infrastructure under its normal ownership.

References: [AWS DR strategies](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html), [Fault Injection Service](https://docs.aws.amazon.com/fis/), [Resilience Hub](https://docs.aws.amazon.com/resilience-hub/), [ARC](https://docs.aws.amazon.com/r53recovery/), [Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/).
