# 15. Machine learning, generative AI, and intelligent applications

[Roadmap index](README.md)

**Prerequisites:** [Python, statistics, and software foundations](01-prerequisites.md), [IAM](04-iam-and-identity.md), [storage](07-storage-and-backup.md), [security](09-security-and-compliance.md), [serverless integration](10-serverless-and-integration.md), [delivery](13-devops-and-delivery.md), and [data engineering](14-data-engineering-and-analytics.md).

**Outcome:** Choose an appropriate AI approach, build a small evaluated application, operate an ML lifecycle, and explain quality, security, latency, and cost tradeoffs. A model invocation is the starting point; a trustworthy production system includes data, authorization, evaluation, monitoring, and recovery.

## 1. Frame the problem and establish a baseline

Write down the decision or task, intended users, allowed inputs, desired outputs, unacceptable failures, available data, and a measurable success criterion. Compare a simple rule or search baseline before introducing machine learning. A deterministic validation rule may solve a problem more reliably than a generative model.

Learn supervised learning, unsupervised learning, classification, regression, clustering, forecasting, ranking, embeddings, and generative modeling. Understand features, labels, training, validation, testing, overfitting, regularization, sampling bias, data leakage, and uncertainty. For time-dependent data, random splitting can leak future information; use a split that reflects the actual prediction situation.

Choose evaluation metrics according to business consequences. Accuracy can hide failure on a rare class. Learn precision, recall, F1, confusion matrices, ROC/PR curves, calibration, and threshold choice for classification; MAE/RMSE and segment-level error for regression; and ranking/retrieval metrics for search. Evaluate important user and data segments, not only an aggregate score.

For generative tasks, distinguish factual correctness, groundedness, relevance, completeness, format validity, safety, latency, and cost. A fluent answer is not automatically correct. Define when the application should abstain, request more information, or route a case to a qualified reviewer.

## 2. Understand the current AWS AI landscape

| Approach | Main path | Responsibility and fit |
| --- | --- | --- |
| Ready-made task API | Services such as Textract, Transcribe, Polly, Translate, and supported Comprehend/Rekognition features | Evaluate a specialized capability against representative data and verify feature availability |
| Managed foundation-model application | Amazon Bedrock | Select models, build prompts/retrieval/tools, enforce application permissions, and evaluate outputs |
| Custom training or controlled model hosting | Amazon SageMaker AI | Manage data, training configuration, model artifacts, evaluation, deployment, and model operations |
| Integrated analytics/AI workspace | Amazon SageMaker Unified Studio | Collaborate across data and AI with governed projects and underlying services |
| Agent runtime and supporting infrastructure | Amazon Bedrock AgentCore | Host or configure agents, connect tools, manage identity/memory, and observe execution |
| Direct infrastructure | EC2, ECS, EKS, accelerated instances | Maximum control with corresponding responsibility for serving, scaling, security, and utilization |

Amazon **SageMaker AI** refers to the ML development/training/deployment service. The broader **Amazon SageMaker** experience includes Unified Studio and data/AI capabilities. Bedrock and SageMaker AI are complementary: one application might use Bedrock for generation and a custom SageMaker AI model for a specialized prediction. See the [AWS service decision guide](https://docs.aws.amazon.com/decision-guides/latest/decision-guides/bedrock-or-sagemaker.html) and [SageMaker overview](https://docs.aws.amazon.com/next-generation-sagemaker/latest/userguide/what-is-sagemaker.html).

Check feature lifecycle before following a tutorial. AWS's maintenance list includes Kendra, Q Business, and several SageMaker AI features including Ground Truth, Clarify, Debugger, Model Monitor, A2I, and Studio Lab; they must not be assumed available to a new customer. Learn the underlying labeling, fairness, monitoring, and review practices using current alternatives. See the [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html) and [Model Monitor alternatives](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor-availability-change.html).

Bedrock Agents launched in 2023 is now **Bedrock Agents Classic** and closed to new customers on July 30, 2026. Bedrock model inference, Knowledge Bases, and Guardrails are not affected by that change. Use current AgentCore paths for a new agent exercise. See [Agents Classic maintenance guidance](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html).

## 3. Learn the conventional ML lifecycle

Start with a small local experiment and move only the necessary stages to AWS. Version the data snapshot, preprocessing code, feature definitions, algorithm/library versions, hyperparameters, random seeds, and evaluation results. Reproducibility means another person can reconstruct the inputs and procedure; hardware and nondeterministic operations may still introduce numerical variation.

Separate preprocessing learned from training data from transformations applied to validation/test data. Fit scaling, imputation, vocabulary, and feature selection only on the permitted training portion. Watch for target leakage through fields created after the prediction time. Keep a held-out test set that is not repeatedly used to tune choices.

In SageMaker AI, learn processing jobs, training jobs, framework/container selection, script mode, input/output channels, distributed training basics, experiment tracking, model registry, pipelines, and deployment. Use a custom container only when its extra control is justified. Learn how the container receives data/configuration and writes model artifacts. Protect training data and artifacts with narrow roles, encryption, and appropriate network controls.

Understand feature stores when multiple training and serving paths need consistent features. Learn offline versus online retrieval, point-in-time correctness, feature freshness, and training-serving skew. A feature store is useful infrastructure, not a substitute for defining features correctly.

Select inference mode from request shape. Real-time endpoints serve interactive traffic; asynchronous inference decouples acceptance and completion for suitable work; batch transform handles finite datasets; serverless inference serves supported workloads with its own latency and feature constraints. Check compatibility rather than assuming every model, accelerator, or networking feature exists in every mode. See [SageMaker inference options](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html).

Build model operations around registered versions and repeatable pipelines. Compare candidate and baseline models, document approval criteria, deploy gradually where appropriate, and retain a recovery option. Monitor service health, input quality, prediction distributions, and actual outcome quality when labels arrive. Drift is a signal to investigate; automatically retraining and promoting every drifted model can amplify errors.

## 4. Build with foundation models deliberately

Learn tokens, context windows, input/output limits, sampling parameters, structured outputs, tool calling, streaming responses, embeddings, multimodal inputs, and model versioning. A larger context window does not guarantee that the model uses every relevant detail correctly. Tokenization and capabilities differ by model.

Use the supported Bedrock API/interface for the chosen model, such as Converse where available. Keep model identifiers and inference configuration versioned. Verify regional availability, required model access or provider terms, throughput quotas, supported input types, and current lifecycle. Do not assume one model's request schema applies unchanged to another. See [model access documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html).

Use clear task instructions, input delimiters, output schemas, and representative examples. Validate generated JSON against a schema and validate business fields independently. A model-selected tool argument remains untrusted input. Retry only appropriate failures with bounded attempts; endlessly asking a model to repair its answer can be expensive and still wrong.

Compare prompting, retrieval-augmented generation (RAG), fine-tuning, and training from scratch. Prompting is a low-cost starting point. RAG brings relevant external evidence into a request and can support citations. Fine-tuning adapts behavior or specialized patterns but is not a reliable substitute for maintaining a fresh, access-controlled knowledge source. Training from scratch requires substantial data, expertise, and compute; do not make it a beginner prerequisite.

## 5. Build and evaluate retrieval

A RAG pipeline includes document acquisition, parsing, chunking, metadata, embedding/indexing, retrieval, optional reranking, context assembly, generation, and citation presentation. Preserve source identity and version through every stage. Evaluate tables, scanned PDFs, and images separately because parsing errors can become confident answer errors.

Experiment with chunk size and overlap based on document structure. Keep headings and useful metadata. Compare lexical, vector, and hybrid retrieval when supported. Tune retrieval depth and reranking with a labeled question set rather than intuition. Too much retrieved context can increase cost, distract the model, or expose irrelevant sensitive material.

Enforce tenant and document permissions before returning content to the model, not merely when rendering citations. Derive filters from authenticated user context on the server. Recheck access when documents change ownership or users lose access. Plan document deletion, reindexing, stale embeddings, and invalidation of cached answers.

Current Bedrock Knowledge Bases provides managed and customer-managed approaches. They differ in infrastructure ownership, connectors, permissions, and customization. Inspect the selected type's supported sources and authorization behavior; do not assume every connector or vector store provides the same document-level controls. See [Knowledge Bases documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html).

Separate retrieval evaluation from generation evaluation. If the correct evidence never appears in retrieved results, changing answer wording will not fix the root cause. Measure retrieval recall/precision, answer correctness, citation support, unsupported claims, and appropriate abstention. Include unanswerable questions and intentionally conflicting documents.

## 6. Secure tools, agents, and data boundaries

An agent combines a model with tools, execution state, and an iterative decision process. Start with a deterministic workflow or a small tool set; add autonomous planning only where it improves the task. Set maximum steps, time, tool calls, and spending. Make timeouts and cancellation release resources predictably.

Learn AgentCore Runtime, Gateway, Identity, Memory, and observability concepts, selecting only required components. AgentCore can support different frameworks and models; the runtime is not a replacement for application authorization or business policy. Review current deployment, session, and regional constraints in the [AgentCore developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html).

Treat retrieved documents, tool results, websites, and uploaded files as untrusted data. They may contain prompt injection asking the model to ignore instructions, reveal secrets, or call tools. Enforce tool permissions outside the model, validate arguments, constrain allowed destinations, and separate read operations from state-changing operations. Require a deliberate authorization step for consequential actions, with a reviewable description of the exact proposed effect.

Scope memory and caches to the correct user/tenant and set deletion/retention behavior. Never use one shared conversation store without an isolation design. Avoid placing secrets in prompts. Log enough information for debugging and audits while minimizing sensitive content. Apply sanitization, access controls, and retention to prompts, retrieved passages, tool traces, and outputs.

Bedrock Guardrails can support configured content, sensitive-information, and other checks, but no guardrail makes output guaranteed correct or replaces IAM and application permissions. Test both false positives and missed cases using your own evaluation set. See [Guardrails documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html).

If using cross-Region inference, inspect the inference profile's actual routing scope and organizational restrictions. A geographic profile and a global profile have different processing-location implications; the source API endpoint alone does not prove where inference runs. See [cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html).

## 7. Measure quality and economics together

Maintain a versioned evaluation set with normal tasks, edge cases, ambiguous requests, malicious retrieved text, unauthorized-data requests, unsupported questions, and representative languages/modalities. Use deterministic checks for formats and known facts, human review for nuanced criteria, and model-based judging only with a validated rubric and awareness of judge bias. Repeat stochastic tests enough to understand variation.

Track cost per successful task rather than only cost per token. A useful estimate is:

```text
Task cost = model input + model output + embeddings + retrieval/reranking
          + tool/runtime calls + storage/logging + retries/evaluations
```

For provisioned endpoints, add idle capacity and utilization. Compare smaller models, routing, batching, caching, shorter context, and supported quantization or accelerator options using measured quality. Learn GPU memory/throughput, Trainium and Inferentia at an advanced level, including software compatibility and operational tradeoffs. The fastest or cheapest individual call may create more retries or incorrect work overall.

Track time to first token, total completion time, tokens per task, rate limits, queue delay, tool latency, and completion quality. Configure quotas, application budgets, concurrency, and alarms. No static roadmap can promise a free inference or a fixed model price; check current service and model pricing.

## 8. Lab A: evaluated knowledge assistant

Use a small collection of documents you own, containing synthetic policies and no personal information. Include contradictory versions and one document containing a harmless test instruction attempting to redirect the assistant.

1. Create a question set with expected evidence, allowed answers, unanswerable cases, and tenant-specific access expectations before tuning the system.
2. Run a plain model baseline with a bounded number of requests. Record model/version, settings, latency, token use, and rubric scores.
3. Build a local retrieval baseline, then optionally use Bedrock Knowledge Bases or a chosen managed store. Preserve document IDs, versions, and access metadata.
4. Generate answers with source citations and an explicit insufficient-evidence path. Validate structured output and enforce authorization before retrieval.
5. Compare retrieval configurations and two suitable available models using the same evaluation set. Avoid changing several factors at once.
6. Add one read-only tool with narrow permissions and server-side argument validation. Set a maximum tool-call count and request budget.
7. Test prompt injection, deleted documents, revoked access, stale versions, throttling, tool failure, and an unavailable model.

**Verify:** Answers cite actual supporting passages; unauthorized content never reaches the model; unsupported questions abstain appropriately; injected document instructions cannot authorize tool actions; and failures produce bounded, explainable behavior. Save evaluation results rather than only a convincing demo transcript.

**Cost and cleanup:** Include inference, embeddings, parsing, indexing, vector-store minimum capacity where applicable, reranking, agent runtime, and logs. Delete dedicated knowledge bases and their underlying stores separately where required, remove ingestion sources/objects and caches, stop agents/schedules, and inspect retained logs and secrets. Deleting the chat UI alone does not stop backend charges.

## 9. Lab B: repeatable predictive model

Choose a small licensed public or synthetic tabular dataset. Define a simple baseline, a leakage-safe split, and an evaluation metric. Train locally, then optionally run a short SageMaker AI training job with narrow data permissions. Track preprocessing and model artifacts, register the candidate, and run batch predictions to avoid leaving an endpoint running unnecessarily.

**Verify:** Reproduce the metrics from recorded inputs; test missing/out-of-range features; compare segments; confirm feature order and preprocessing consistency; and join delayed labels to predictions using an inference identifier. Introduce a shifted synthetic batch and demonstrate an alert or review report without automatically promoting a new model.

**Cleanup:** Stop active jobs and delete dedicated endpoints, endpoint configurations, models, temporary IDE applications, schedules, model artifacts, containers, and logs as appropriate. Check persistent volumes and tracking infrastructure. A stopped notebook or closed browser tab does not prove all ML resources stopped.

## Mastery checklist

- [ ] I can justify rules, a specialized API, Bedrock, or a custom ML model for a task.
- [ ] I can explain leakage-safe splits, baselines, metrics, and uncertainty.
- [ ] I can reproduce a training/evaluation run from versioned inputs.
- [ ] I can distinguish retrieval quality from generation quality.
- [ ] I can enforce document and tool permissions outside the model.
- [ ] I have tested prompt injection, unsupported questions, and model/tool failure.
- [ ] I can explain per-task cost, idle capacity, and cleanup.
- [ ] I can check model and feature lifecycle before adopting an old tutorial.

## Official references

- [SageMaker AI developer guide](https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html).
- [Bedrock user guide](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html).
- [Bedrock evaluation capabilities](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html).
- [SageMaker Pipelines](https://docs.aws.amazon.com/sagemaker/latest/dg/pipelines.html).
- [AWS Well-Architected Machine Learning Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/welcome.html).
- [AWS responsible AI](https://aws.amazon.com/ai/responsible-ai/).

**Review date:** 2026-10-18. Models, SDKs, supported integrations, and individual AI features change quickly; verify current access, lifecycle, licensing, regional routing, and pricing before deployment.
