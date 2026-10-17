# Tutorial 03.2 — Turn requirements into a defensible architecture

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../03-cloud-foundations.md)

**Mode:** local worked architecture exercise. No AWS deployment. **Result:** requirements, two alternatives, a decision record, and cost inputs. Reviewed 2026-10-18.

## 1. Start with a specific workload

You are designing a fictional document portal for 200 users. Each user uploads ten 2 MB files per working day. An upload should become visible in the user's history promptly; thumbnail generation can be asynchronous. Files are private to their owner. Keep original files for one year, but make deletion requirements explicit rather than assuming every file must remain forever.

Use these **exercise assumptions**, not production promises: 22 working days monthly; peak traffic ten times the daily average; API p95 response target of 500 ms excluding file transfer; service availability target of 99.9%; RTO four hours; RPO one hour. Ask the fictional business owner to resolve the conflict between one-year retention and a user's deletion request.

Calculate 2,000 uploads per working day and about 4 GB of new original content per working day using decimal units. Monthly input is about 88 GB before thumbnails, versions, logs and backups. Record that assumption so the later bill is explainable.

## 2. Define measurable success

Write five acceptance statements:

1. A signed-in user can upload a permitted small file and see its metadata.
2. Another user cannot read the file or its private metadata.
3. A repeated thumbnail event does not create duplicate history records.
4. An operator can recover a documented point in time within the exercise RTO.
5. Cost can be allocated to environment and useful workload units.

Availability must identify the measured operation. A homepage responding while upload creation fails is not a successful upload service. Durability concerns retained data, while availability concerns access now; an archive can preserve data but fail an immediate-retrieval requirement.

Choose latency percentiles rather than an average alone. Ten extremely slow uploads can disappear inside a pleasant average. Include correctness: returning another person's file quickly still fails the system requirement.

## 3. Build alternative A: managed event-driven components

Draw authenticated client → API → metadata database, and client → private object upload. Draw the object/event path to a queue or event integration, then a worker that creates thumbnails and updates metadata. Name Lambda as a candidate worker when execution duration and resource needs fit; use a container/batch alternative for longer processing.

The API issues a scoped upload capability only after authorization. The worker uses its own role. A failure destination retains processing failures for investigation and replay. Keep original files authoritative; thumbnails can be regenerated. Avoid storing the only copy on a worker's local disk.

This architecture reduces host administration but still needs quotas, concurrency limits, retry behavior, deployment, logging and on-call ownership. “Serverless” is not a substitute for those decisions.

## 4. Build alternative B: EC2 application and workers

Draw a load balancer in front of replaceable application instances across two AZs, private durable storage, and a bounded worker pool reading the same work queue. Identify the AMI, patching owner, instance role, security groups, bootstrap dependencies and scaling signal.

The important difference is operating responsibility and workload control, not whether servers physically exist. EC2 gives OS/runtime flexibility but adds image, patch and capacity management. A managed database changes who operates its host; it does not choose your schema or remove the need to test restoration.

Add a third procurement alternative: a suitable managed SaaS document product. It may satisfy the business outcome with less custom engineering, but evaluate user lifecycle, data export, retention, integrations, contract and cost. This makes the infrastructure/platform/application operating-model distinction concrete.

## 5. Compare scale and elasticity

Suppose the worker processes one file per second. A burst of 600 files takes roughly ten minutes on one worker ignoring overhead. Six equivalent workers could reduce the ideal processing period toward 100 seconds, but shared storage, downstream writes and startup time may prevent ideal scaling.

Vertical scaling means giving a worker more resources; horizontal scaling means adding workers. Elasticity changes capacity with demand. Define backlog age or backlog per worker as a candidate signal rather than assuming CPU reveals unmet demand.

Record minimum/maximum capacity and the time needed to become useful. A scaling maximum of two workers is a deliberate limit; it cannot meet the ideal six-worker assumption. Conversely, unbounded workers may overload the metadata database.

## 6. Build the cost worksheet

Use columns `dimension`, `quantity`, `current regional rate`, `monthly estimate`, `assumption`, and `owner`. Include execution time, instance idle time, object storage, versions, requests, thumbnail storage, database capacity, load balancing, transfer, public IPv4, NAT/endpoints where needed, logs, snapshots and support/licensing.

Do not enter remembered prices. Open the current pricing pages and calculator for the intended Region when converting this exercise into a budget. Count both normal operation and recovery copies. A budget alert is a notification, not a universal stop switch.

Compute a unit-cost denominator: completed uploads per month, or active users per month. If costs rise while successful uploads remain flat, investigate retries, idle infrastructure, growing retention and telemetry volume. Consumption billing improves flexibility but does not automatically make the design cheaper than a simpler product.

## 7. Review six architectural dimensions

For operational excellence, name deployment/rollback and an on-call runbook. For security, test cross-user denial and separate roles. For reliability, demonstrate restoration and duplicate processing. For performance efficiency, measure the queue and query bottlenecks. For cost optimization, inspect idle/retained resources and unit cost. For sustainability, reduce unnecessary work, oversized capacity and duplicate storage consistent with the business requirement.

Turn each review observation into a concrete action. “Improve security” is weak; “deny user A's download request for user B's object and record the observed result” is testable.

## 8. Write the decision and implementation sequence

Select one alternative under the exercise assumptions. State why, what was rejected, and which measurements might reverse the decision. Do not claim multi-Region operation is mandatory: it adds replication, routing and recovery complexity, and the assumed RTO/RPO may be achievable more simply.

Sequence work as identity → private storage → metadata model → upload API → bounded processing → telemetry → deployment → recovery. For each step identify the next evidence gate. Use a decision record that can change when actual traffic replaces assumptions.

## Verify and finish

**Negative review:** remove the queue, double file size, or demand RPO zero. Explain which assumptions break and revise the diagram/cost model. A diagram with two database icons does not prove transaction correctness or recovery.

**Troubleshooting:** if alternatives appear identical, compare responsibility, startup time, concurrency, state and cost dimensions rather than adding more product names. If requirements conflict, record the unresolved business decision instead of silently inventing an answer.

**Cleanup/cost:** no cloud resources were created. Preserve the decision record, calculations and test plan as the deliverable. Sources: [Well-Architected pillars](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html), [AWS Pricing Calculator](https://calculator.aws/), [shared responsibility](https://aws.amazon.com/compliance/shared-responsibility-model/).
