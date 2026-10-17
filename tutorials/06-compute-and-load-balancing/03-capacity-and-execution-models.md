# Tutorial 06.3 — Select compute and prove a scaling plan

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../06-compute-and-load-balancing.md)

**Mode:** worked capacity and cost exercise, with optional measurements from earlier labs. **Result:** execution-model decision, scaling worksheet and purchasing rationale. Reviewed 2026-10-18.

## 1. Classify four jobs

Use these synthetic workloads: an always-on HTTP API; a thumbnail event taking two seconds; a scientific job taking three hours; and a licensed Windows application requiring a specific OS configuration. For each record duration, startup tolerance, memory/CPU, state, hardware, network, concurrency and team operating skill.

Choose a candidate execution model and a rejected alternative. The API could run on EC2 or containers; the thumbnail may fit Lambda; the scientific job may fit Batch over suitable compute; the licensed application may need EC2/dedicated licensing arrangements. These are hypotheses to verify, not automatic product rules.

A serverless function has bounded execution and concurrency characteristics. Splitting a long computation into arbitrary invocations can add checkpoint and orchestration complexity. Conversely, keeping a whole VM idle for rare short events can create unnecessary cost.

## 2. Compare containers and managed hosting

For the API, compare ECS/EKS on EC2 with Fargate. Write who operates worker nodes, container images, workloads, identities, networking and upgrades in each case. Kubernetes portability does not remove cluster operations. Fargate removes host management for supported workloads but still requires resource sizing and application controls.

Evaluate Elastic Beanstalk when its application-platform model fits the team's requirements, and Lightsail for a simpler hosting scope. Document platform/runtime support, customization boundaries, deployment rollback and migration path. App Runner has a maintenance notice; do not make it a new mandatory exercise simply because an older course recommends it.

For Batch, define a job queue, job definition, retry policy and output location. Queueing and scheduling are useful for the three-hour workload; the job must still checkpoint or restart safely. For Lambda, identify event retry semantics, idempotency and downstream limits rather than using “automatic scaling” as the entire plan.

## 3. Select hardware from the bottleneck

Suppose a measured worker uses 25% CPU but waits on storage most of the time. Doubling vCPU count is unlikely to double output. Record CPU, memory, storage latency/queue depth, network and application throughput together.

Compare general-purpose, compute-optimized, memory-optimized, storage-optimized and accelerated families. Validate native libraries for Arm/Graviton versus x86. A GPU helps only when the application uses it effectively; data transfer and preprocessing may dominate the task.

For burstable instances, study CPU-credit behavior and sustained-load cost/performance. For EBS-heavy workloads, compare volume provisioned performance with the instance's EBS limits. For high-network workloads, examine connection behavior and supported networking features, not only headline bandwidth.

For tightly coupled MPI, design a supported placement/EFA/parallel-filesystem arrangement and compare it with independent Batch jobs. Cluster placement, spread and partition strategies address different goals. Do not promise that adding nodes always reduces runtime; communication and synchronization can dominate.

## 4. Calculate a scaling loop

Assume one worker safely handles four jobs per second. Initial demand is six per second, rising to fourteen for five minutes. With two workers, service rate is eight per second; during the burst, backlog grows by six per second. After five minutes that is approximately 1,800 additional jobs, ignoring startup and other effects.

Four workers provide an ideal sixteen jobs per second. If startup takes two minutes, the queue still accumulates work before new capacity is useful. Draw backlog and oldest-job age over time. Set an explicit maximum so a malformed event storm cannot request arbitrary capacity.

Compare target tracking on backlog per worker with CPU target tracking. Low CPU while waiting on a database can coexist with growing backlog. Step scaling can apply different actions at thresholds, scheduled scaling can prepare for known events, and predictive scaling requires suitable demand history. Choose the simplest signal that tracks useful work.

Warmup prevents premature interpretation of new instances. Health grace periods avoid replacing instances before expected initialization, but excessive values delay fault recovery. Add scale-in cooldown/behavior and a drain/checkpoint rule so capacity reduction does not discard active jobs.

## 5. Add interruption and capacity constraints

For the scientific job, use Spot only if interruption can be handled by restart/checkpointing and replacement capacity. Diversify acceptable instance types/AZs where compatible. Simulate an interruption at minute 120 with a checkpoint at minute 110; the application should resume from the last valid checkpoint rather than assume the whole result exists.

A capacity shortage in the selected AZ is different from reaching an account quota. A Capacity Reservation addresses eligible capacity needs; Savings Plans and Reserved Instances address particular pricing/commitment models with differing capacity implications. Do not describe every discounted purchase as a hardware guarantee.

Use On-Demand for the disposable learning workload. Compare longer-term commitments only after measuring stable use. Dedicated Instances/Hosts may matter for isolation or license conditions; verify the exact software vendor terms and supported configurations rather than assuming tenancy alone resolves compliance.

## 6. Build the complete cost comparison

Create a worksheet with workload-hours, useful utilization, idle capacity, startup/drain overhead, storage, snapshots, image retention, load balancer, public IPv4, NAT/endpoints, transfer, logs and licenses. Use current Region-specific rates when calculating money.

For Lambda/Fargate/Batch, identify their relevant billed dimensions and dependent services instead of comparing only an EC2 hourly price with one function invocation. For each model calculate cost per successful job, including retries and failed work.

Add a downside scenario: traffic is one tenth of forecast; then add an upside scenario: traffic triples. A commitment that looks good only at the forecast can become wasteful. A scaling design that ignores downstream write capacity may become expensive without increasing successful throughput.

## 7. Define operational proof

Collect built-in EC2 metrics and add guest memory/disk measurements with a deliberately configured agent when needed. Built-in metrics do not automatically reveal every process or filesystem condition. Record the application release, test data, measurement interval and saturation symptom.

Define an acceptance test: fourteen jobs per second for five minutes, bounded oldest-job age, no lost/duplicated business effect and recovery after one worker disappears. If you do not deploy a load generator, label the calculations as a capacity model rather than measured evidence.

**Negative check:** halve database write capacity. Your plan should throttle workers or apply backpressure instead of scaling forever. Add failed launch and unavailable-secret scenarios to the replacement test.

## Verify and finish

Pass with one justified execution model per workload, a bottleneck hypothesis, explicit scaling limits and complete cost dimensions. If a benchmark conflicts with expectations, inspect input size, architecture compatibility and shared dependencies before choosing a larger instance.

**Cleanup/cost:** the design has no AWS resources. If using earlier lab measurements, stop the experiment and remove dedicated compute and retained dependencies. Do not purchase Savings Plans, reservations, licenses, circuits or accelerators for this exercise.

Sources: [EC2 guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/), [Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/), [Batch](https://docs.aws.amazon.com/batch/latest/userguide/), [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
