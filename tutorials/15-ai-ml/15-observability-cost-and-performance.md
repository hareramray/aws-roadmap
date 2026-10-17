# 15 — Operate an AI workload and measure cost per successful task

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** local measurement plus optional telemetry on the dedicated AWS application. **Result:** a latency/cost breakdown, an actionable dashboard design, a controlled failure test, and a measured optimization. Prerequisites are [evaluation](14-evaluation-and-responsible-ai.md) and the root [operations chapter](../../16-observability-and-operations.md).

## 1. Define one trace and its stages

Assign a request ID at the API boundary. Carry it through authentication, retrieval, embedding/reranking if performed, generation, tool calls, and response validation. Record model/configuration/index versions so a regression can be tied to a change.

Use spans for actual work, not fabricated parallel stages. Collect total latency, time to first token, retrieval duration, model duration, tool duration, attempts, input/output tokens, outcome, and error category. Protect high-cardinality identifiers from accidentally becoming unbounded metric dimensions.

Keep content logging separate from operational metadata. Raw prompts, retrieved documents, model outputs, and tool results may contain sensitive data. Default to counts, IDs, categories, and timings; enable tightly controlled diagnostic samples only when justified.

## 2. Calculate an understandable unit cost

Use current region/model prices and their actual billing units. Calculate:

```text
request cost = input tokens × input rate + output tokens × output rate
             + embedding + retrieval + reranking + guardrail + tool/runtime cost
             + allocated storage, logging and provisioned idle cost
cost per successful task = total cost / count of tasks meeting the success rubric
```

Normalize rates before multiplying. “Per million tokens” and “per thousand tokens” differ by a factor of one thousand. Count all attempts, including failed calls and model-based evaluation. Credits reduce cash paid for a period; they do not change the underlying unit economics.

Save as `measure_ai.py`. This local calculator requires rates you supply from current pricing; it does not embed an assumed AWS price.

```python
import argparse
import math

parser = argparse.ArgumentParser()
parser.add_argument("--input-rate-per-million", type=float, required=True)
parser.add_argument("--output-rate-per-million", type=float, required=True)
args = parser.parse_args()
if min(args.input_rate_per_million, args.output_rate_per_million) < 0:
    raise SystemExit("Rates cannot be negative")

samples = [
    {"in": 900, "out": 120, "ms": 800, "success": True},
    {"in": 1500, "out": 220, "ms": 1300, "success": True},
    {"in": 700, "out": 80, "ms": 1900, "success": False},
    {"in": 1000, "out": 160, "ms": 950, "success": True},
]
cost = sum((row["in"] * args.input_rate_per_million
            + row["out"] * args.output_rate_per_million) / 1_000_000
           for row in samples)
successes = sum(row["success"] for row in samples)
latencies = sorted(row["ms"] for row in samples)
p95 = latencies[math.ceil(0.95 * len(latencies)) - 1]
print({"model_cost_only": round(cost, 8), "successful_tasks": successes,
       "model_cost_per_success": round(cost / successes, 8) if successes else None,
       "p95_ms_nearest_rank": p95, "sample_count": len(samples)})
```

Use illustrative nonnegative rates to understand the arithmetic, then your real current rates for an estimate. The four synthetic samples yield three successes and nearest-rank p95 of 1900 ms. Four samples are far too few for a credible production tail-latency claim. The printed cost deliberately excludes non-model components; add them before presenting total workload cost.

## 3. Build a dashboard tied to actions

Create panels for request/task success rate, unsupported-answer rate, authorization rejection, p50/p95 total latency, throttling, model/provider failures, tool errors, queue age, token usage, estimated cost, and deployed versions. Link each alert to an owner and runbook.

Use supported Bedrock, AgentCore, CloudWatch, and OpenTelemetry instrumentation for the actual runtime. Distinguish a hosted runtime error from a valid model response that fails your quality rubric. A green HTTP 200 dashboard can hide incorrect answers. [AgentCore observability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html)

For predictive ML, add feature missingness/ranges, schema changes, prediction distribution, serving latency/errors, and delayed outcome metrics. Join predictions with later labels using stable IDs. Distribution drift prompts investigation; it does not automatically justify promoting a retrained model.

## 4. Bound work and test overload

Set per-request token limits, tool-call/step limits, deadlines, input-size limits, tenant quotas, and concurrency limits. Configure bounded retries with backoff/jitter for appropriate transient errors. Multiple layers independently retrying can multiply calls and spending.

Run a small fixed batch of synthetic requests at a known concurrency. Record the load size and stop condition before starting. Increase one dimension gradually within the lab budget. Observe whether queueing or throttling occurs and whether clients receive a controlled retryable response. Do not run an unbounded load generator against a paid model endpoint.

Simulate a slow tool locally and verify cancellation releases resources. Simulate a model error and verify retries stop at the configured bound. A fallback model requires its own permission, cost, quality, and data-location review; do not silently route sensitive data to an unreviewed alternative.

## 5. Optimize one measured bottleneck

Choose a hypothesis such as “irrelevant retrieved chunks increase model tokens without improving correctness.” Compare retrieval depths 3 and 8 on the same evaluation set. Record quality, token count, latency, and complete cost. Select the smaller context only if the evidence supports it.

Other experiments include a smaller suitable model, model routing by task, batching for noninteractive work, prompt/context caching where supported, shorter output contracts, reranking only when beneficial, and lower idle endpoint capacity. Cache keys must include tenant/authorization context, source version, and configuration as needed; a cheap cache that leaks another user's answer is unacceptable.

For self-hosted models, inspect memory, throughput, batch size, quantization compatibility, and accelerator utilization. An inexpensive instance with long queues or low task quality may produce higher cost per success than a more capable option. Continue with [accelerated compute](18-accelerated-compute-and-unified-studio.md).

## 6. Recovery and cleanup

Create a runbook for model unavailable, throttling, wrong index version, faulty prompt, data leak suspicion, and runaway cost. Include version rollback, traffic limits, session/cache invalidation, evidence preservation, and user impact verification.

Stop the test load and remove dedicated endpoints, provisioned throughput, runtime resources, scheduled evaluation jobs, vector stores, and unused artifacts when appropriate. Set log/trace retention and confirm dashboards do not reference deleted resources misleadingly. Model invocations, runtime time, indexes, storage, and security/telemetry can all contribute charges.

**Completion:** provide a before/after experiment report and explain why the selected change improves useful task outcomes, not merely one isolated metric. Use current [Bedrock pricing](https://aws.amazon.com/bedrock/pricing/) and [SageMaker pricing](https://aws.amazon.com/sagemaker/ai/pricing/) for real estimates.
