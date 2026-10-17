# AWS AI, machine learning, and generative AI tutorials

[All tutorials](../README.md) · [Original AI roadmap](../../15-ai-ml-and-generative-ai.md)

These 18 tutorials turn the AI roadmap into worked lessons. Start with local synthetic-data exercises, then choose the optional AWS stages that fit your authorized sandbox and budget. A local test proves its stated behavior; it does not prove real AWS permissions, model quality, or regional availability.

## Learning sequence

| Order | Tutorial | Practical result |
|---|---|---|
| 01 | [ML foundations](01-ml-foundations.md) | Frame a problem, calculate metrics, compare a baseline, and detect misleading accuracy |
| 02 | [Data preparation and feature engineering](02-data-preparation-and-feature-engineering.md) | Prepare reproducible splits/features without leakage |
| 03 | [SageMaker training and experiments](03-sagemaker-training-and-experiments.md) | Package training inputs/code, inspect jobs, and preserve artifacts/results |
| 04 | [SageMaker inference](04-sagemaker-inference.md) | Choose and validate batch, asynchronous, serverless, or real-time serving |
| 05 | [MLOps pipelines and registry](05-mlops-pipelines-and-model-registry.md) | Gate promotion, version artifacts, and plan monitoring/rollback |
| 06 | [Bedrock models and inference](06-bedrock-models-and-inference.md) | Make a controlled model call and inspect usage/errors |
| 07 | [Prompts, structured output, and tools](07-prompting-structured-output-and-tool-calling.md) | Validate outputs and enforce a bounded tool contract |
| 08 | [Fine-tuning and model selection](08-fine-tuning-and-model-selection.md) | Compare prompting, retrieval, adaptation, and training from scratch |
| 09 | [Embeddings, chunking, and local RAG](09-embeddings-chunking-and-local-rag.md) | Run a retrieval baseline with tenants, versions, citations, and deletion tests |
| 10 | [Bedrock Knowledge Bases](10-bedrock-knowledge-bases.md) | Ingest synthetic documents and inspect filtered retrieval/generation |
| 11 | [AgentCore Runtime](11-agentcore-runtime.md) | Develop, preview, host, test, and remove a minimal bounded agent |
| 12 | [Tools, identity, and memory](12-agent-tools-identity-and-memory.md) | Enforce tool authorization and isolate/revoke memory access |
| 13 | [Guardrails and AI security](13-guardrails-and-ai-security.md) | Test content policies, injection boundaries, privacy, and failure controls |
| 14 | [Evaluation and responsible AI](14-evaluation-and-responsible-ai.md) | Build fixtures, rubrics, segment checks, and a release decision |
| 15 | [Observability, cost, and performance](15-observability-cost-and-performance.md) | Measure complete task quality/latency/cost and optimize one bottleneck |
| 16 | [Specialized AI APIs](16-specialized-ai-apis.md) | Compare translation, language analysis, OCR, speech, and image capabilities |
| 17 | [AI capstone](17-ai-capstone.md) | Assemble a local assistant and add AWS components incrementally |
| 18 | [Accelerators and Unified Studio](18-accelerated-compute-and-unified-studio.md) | Plan GPU/Trainium/Inferentia benchmarks and governed workspace isolation |

## Coverage mapping

| Original AI topic and subtopics | Tutorials |
|---|---|
| Problem framing, simple rules/search baseline, supervised/unsupervised learning, classification/regression/clustering/forecasting/ranking | 01, 08 |
| Features, labels, train/validation/test, overfitting, regularization, sampling, leakage, uncertainty | 01–02 |
| Precision, recall, F1, confusion matrix, ROC/PR, calibration, MAE/RMSE, retrieval metrics | 01, 14 |
| Specialized APIs, Bedrock, SageMaker AI, Unified Studio, AgentCore, direct compute | 03–06, 11, 16, 18 |
| Service/model/feature lifecycle and current-account availability | 03–06, 08, 11, 16, 18; global start/source-maintenance guides |
| Processing/training jobs, containers/script mode, input/output channels, experiments, artifacts, distributed training | 02–03, 18 |
| Feature stores, point-in-time correctness, offline/online features, freshness, training-serving skew | 02, 05 |
| Real-time, async, batch, serverless inference and compatibility | 04 |
| Registry, pipelines, promotion, rollback, monitoring/drift/delayed labels | 05, 14–15, 17 |
| Tokens, context, sampling, streaming, multimodal inputs, model/version selection | 06–08, 16 |
| Prompt contracts, output schemas, tool calling, validation, bounded retries | 07, 12–15 |
| Prompting versus RAG versus fine-tuning versus pretraining | 08–10, 18 |
| Parsing, chunking, metadata, embeddings, lexical/vector/hybrid retrieval, reranking, citations | 09–10, 16 |
| Knowledge Base storage/ownership, connectors, permissions, source updates/deletes, cached answers | 09–10 |
| Tenant/document authorization, identity propagation, revoked access | 09–13, 17 |
| Agent Runtime, Gateway, Identity, Memory, session isolation, tools, observability | 11–12, 15 |
| Prompt injection, sensitive data, guardrails, unsafe output, cross-region inference | 12–14 |
| Normal/adversarial/unanswerable cases, deterministic checks, human rubric, model judges, responsible AI | 13–14 |
| Token/runtime/store/idle cost, task success, latency, concurrency, throttling, caching, hardware optimization | 15, 18 |
| Knowledge-assistant lab and predictive-model lab | 17 and 01–05 |
| AI portfolio, interview explanations, readiness and certification route | 17 and [study/interview practice](../23-practice-and-reference/03-study-certification-and-interviews.md) |

## Routes

For conventional ML, follow 01–05, 14–15, then the predictive-model companion in 17. For generative AI applications, follow 01–02, 06–10, 13–15, and 17. Add 11–12 for agents after the deterministic workflow works. Use 16 for task-specific services and 18 for advanced infrastructure/workspace needs.

**Completion gate:** preserve runnable local fixtures, evaluation evidence, permission/data-flow diagrams, a measured or clearly labeled estimated cost model, and an exact cleanup ledger for any AWS stages you actually execute.
