# 18 — Evaluate accelerators and a governed AI workspace

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** worked planning/benchmark design with optional short-lived authorized deployments. **Result:** an accelerator selection record and a project-isolation experiment. These are advanced branches; neither expensive hardware nor a full enterprise workspace is required for the earlier AI tutorials.

## 1. Size the workload before choosing hardware

Write down model parameter count, parameter precision, sequence length, batch size, concurrency, latency target, training versus inference, and supported framework. Separate parameter memory from activations, attention/KV cache, optimizer state, gradients, temporary workspaces, runtime overhead, and replicated copies.

A rough parameter-only estimate is `parameters × bytes per parameter`. For a hypothetical 3-billion-parameter model at two bytes per parameter, parameters alone are about 6 billion bytes, approximately 5.6 GiB. That does not prove the workload fits in a 6 GiB device: serving and training need additional memory. Quantization can reduce some storage/compute demands but requires compatibility and quality evaluation.

For training, compare data parallelism, model/tensor parallelism, pipeline parallelism, and sharded state. Explain which data moves between devices and how communication can become the bottleneck. A larger number of devices can increase overhead when batches or communication patterns are unsuitable.

## 2. Build a candidate table

Compare CPU, GPU-backed instances, Trainium, Inferentia, and managed Bedrock/SageMaker options against the actual model/task. Include compiler/framework support, precision/quantization, memory, batch/sequence constraints, startup/compilation time, regional capacity, quotas, operational skills, and total cost.

Use current official hardware/Neuron documentation to verify supported model/runtime combinations. Do not assume a PyTorch script runs unchanged on every accelerator. Keep the model, weights/license, framework, driver/runtime, compiler, and container versions together. [AWS Neuron getting started](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/quick-start/index.html)

## 3. Design a bounded benchmark

Prepare a fixed synthetic or licensed dataset and a known baseline metric. Define warm-up separately from measured work. For inference, measure quality, throughput, time to first token where relevant, p50/p95 latency, memory, and cost at realistic concurrency. For training, measure time to a specified quality target rather than only examples per second.

Use at least two request sizes and record batch/sequence settings. Compare equivalent output quality and semantics. Include failed requests and compilation/startup in the operational cost model; a warm steady-state number may not represent an intermittently used workload.

If deploying, choose the smallest suitable approved instance/configuration, a fixed experiment duration, a shutdown owner, and a supported image. Confirm region quota/capacity before allocating. Use private data paths and a workload role. Install only the documented compatible runtime stack, then run one official minimal check before loading your workload.

## 4. Diagnose the result

If memory fails, inspect all allocations rather than assuming parameter size is the whole footprint. If throughput is poor, inspect utilization, CPU preprocessing, storage/network input, batch shape, synchronization, and compilation. If quality changes after quantization, evaluate the same held-out cases before accepting the performance improvement.

Calculate cost per successful inference and, for training, cost per completed validated model run. Include idle time, attached storage, checkpoints, transfer, and engineering complexity. Describe how the chosen runtime would be patched, autoscaled, monitored, and recovered when hosted on EC2/ECS/EKS.

Stop/terminate only the dedicated lab compute, then inspect volumes, snapshots, images, caches/artifacts, and logs. A stopped notebook or completed training process is not proof every underlying resource stopped billing.

## 5. Understand Unified Studio boundaries

SageMaker AI's training/hosting service and the broader SageMaker Unified Studio workspace are different layers. A Unified Studio project groups people, tools, data, and resources for a use case, with capabilities determined by its project profile and domain setup. Do not infer that opening a notebook grants every underlying data permission. [Unified Studio projects](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/projects.html)

For a design-only exercise, draw a domain containing two projects: `support-ai` and `finance-analysis`. Specify owners, contributors, approved data, execution roles, network policy, model access, and budget ownership. Identify which catalog/data access approvals are needed and how an owner removes a collaborator.

## 6. Optional project-isolation experiment

In an existing authorized training domain, create a dedicated project using a profile that supplies only the required features. Add two test principals with different roles through the supported membership workflow. Use synthetic data in a private governed location. Record the actual project/domain IDs, provisioned resources, and execution identities.

Open the supported notebook/editor environment, inspect its runtime identity, read one approved dataset, and perform a local computation such as counting rows. Attempt access to a second test dataset that is deliberately not granted; record the denial. Distinguish project membership, catalog subscription/approval, storage permissions, encryption-key permissions, and network reachability.

Remove the second member and confirm the intended access-revocation behavior, including existing sessions according to their documented lifetime. Do not assume revoking UI membership instantly invalidates every previously issued temporary credential. [Create a project and add members](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/getting-started-create-a-project.html)

## 7. Reproducibility and workspace cleanup

Version code/configuration and record dataset snapshots, package versions, environment/profile, experiment parameters, and model artifacts. A notebook that only runs because of hidden state in earlier cells is not reproducible. Restart the kernel and run from the beginning on the synthetic dataset.

Close/stop the specific compute applications and inspect persistent storage, project-created resources, jobs, endpoints, repositories, and logs. Follow the documented project deletion/decommission workflow only for your dedicated project; do not delete the shared domain. Record retained data and its owner.

**Completion:** submit one accelerator decision with measured or clearly labeled estimated values, a benchmark protocol, and an identity/data-boundary diagram for a governed workspace. State which advanced steps were designs and which were actually executed.
