# 10. Serverless applications and service integration

[Roadmap index](README.md)

**Prerequisites:** [account and tooling](02-account-and-tooling.md), [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md), [databases](08-databases-and-caching.md), and [security](09-security-and-compliance.md). Continue with [infrastructure as code](12-infrastructure-as-code.md) and [delivery](13-devops-and-delivery.md) to make the application reproducible.

**Outcome:** Design an API and event pipeline that survives retries, duplicate events, slow dependencies, and partial failure. Explain where messages wait, who retries them, how access is authorized, and how to prove work completed.

## 1. Start with execution and failure models

Serverless means delegating more infrastructure management to the platform. Capacity, permissions, dependencies, observability, recovery, and cost remain application responsibilities. Lambda, API Gateway, SQS, EventBridge, Step Functions, DynamoDB, and managed containers solve different parts of this problem; they are not interchangeable deployment styles.

Distinguish three invocation paths before writing a handler:

- **Synchronous:** The caller waits for a response and normally owns retry decisions. An HTTP timeout does not prove the server performed no work. Supply an idempotency key for operations such as creating an order.
- **Asynchronous:** An event is accepted for later processing. Acceptance is different from business completion. Know the event age limit, retry policy, failure destination, and replay procedure.
- **Poll based:** An event source mapping reads queues or streams and invokes Lambda. Batch acknowledgment, checkpoints, visibility, concurrency, and failure isolation depend on the source.

Create a failure matrix for each boundary: rejected before acceptance, accepted but response lost, handler error, timeout after a side effect, and poisoned input. Record whether each case can cause loss, duplication, delay, or reordering. This exercise is more valuable than memorizing a service diagram.

## 2. Learn Lambda beyond the first function

Understand handlers, supported runtimes, deployment packages and container images, execution environments, initialization, warm reuse, ephemeral storage, memory/CPU allocation, timeouts, versions, aliases, and configuration. Keep durable state outside the process. Reused global clients can improve efficiency; reused request-specific mutable state can leak data across invocations.

Use environment variables for nonsecret configuration and a managed secret store for secrets. Apply a narrow execution role. Separate the permission to invoke a function from the permissions its code receives. Resource policies, execution roles, VPC controls, and application authorization each answer different questions.

Measure duration distributions and memory use before tuning. More memory can shorten execution sufficiently to reduce cost, but measure the actual workload. Reserved concurrency caps and reserves capacity for a function; provisioned concurrency keeps execution environments initialized for suitable workloads. Account concurrency, per-function settings, event-source limits, downstream database limits, and burst behavior interact. Check current quotas rather than copying numbers from a tutorial.

A Lambda function attached to a VPC does not receive internet access merely because a selected subnet is public. Plan routes, NAT where needed, or private endpoints, and account for their cost. Reuse database connections cautiously and evaluate RDS Proxy for compatible relational workloads. Limit fan-out so a quickly scaling function cannot overwhelm a slowly scaling dependency.

Learn structured logs, correlation IDs, business metrics, distributed tracing, and log retention. Record order IDs and outcomes without logging credentials or entire sensitive payloads. Use the maintained Powertools utilities for your chosen language when their logging, tracing, validation, or idempotency behavior matches the design.

## 3. Build the API boundary

Compare API Gateway HTTP APIs, REST APIs, and WebSocket APIs using the features your application needs: authentication, integration types, request validation, transformations, throttling, caching, private access, connection behavior, and operational controls. Feature support differs; select an API type after checking the current matrix. AppSync is an additional path for GraphQL and subscription-oriented applications; learn resolvers, data sources, field authorization, and N+1 access patterns before adopting it.

Define an OpenAPI contract or equivalent schema. Validate types, lengths, allowed values, content types, and payload sizes. Use stable error codes and a documented pagination strategy. Keep an API version compatible while clients migrate. Return an accepted response and a job identifier when work outlasts an interactive request.

Authentication establishes identity; authorization decides which records and operations that identity may access. Learn IAM authorization, JWT validation, Cognito user pools, and Lambda authorizers as appropriate. Validate issuer, audience, expiry, and authorization context. API keys identify consumers for usage management; they are not sufficient user authentication. CORS controls browser behavior and is not a permission boundary against nonbrowser callers.

## 4. Choose the right messaging pattern

| Service or pattern | Primary job | Design questions |
| --- | --- | --- |
| SQS standard | Durable work buffer with competing consumers | Can duplicates and out-of-order processing be tolerated? How old can work become? |
| SQS FIFO | Ordered processing within message groups with producer deduplication features | What is the grouping key? Can one blocked group stall important work? |
| SNS | Push fan-out to subscribers | Does each consumer need its own SQS queue, filter, retry policy, and DLQ? |
| EventBridge event bus | Route business or service events by content | What is the event contract, target failure behavior, archive, and replay policy? |
| EventBridge Pipes | Connect a supported source to a target with optional filtering and enrichment | Which acknowledgment and retry rules are inherited from the source? |
| EventBridge Scheduler | Schedule individual or recurring invocations | What are the time zone, flexible window, retry, and missed-run expectations? |
| Kinesis Data Streams / MSK | Retained ordered event logs and multiple consumers | Is ordering needed per partition? How will consumers catch up or replay? |
| Amazon MQ | Managed brokers for compatible messaging protocols | Is protocol compatibility needed for a migration? Who owns broker sizing and client behavior? |

An event describes something that happened, such as `OrderAccepted`; a command requests an action, such as `ReserveStock`. Use versioned schemas containing an event ID, event type, occurrence time, producer, correlation ID, and business key. Store large payloads in an appropriately secured object store and pass a reference plus integrity metadata.

SQS FIFO deduplication is not an end-to-end guarantee that an external side effect occurs only once. Consumers, timeouts, and acknowledgment failures still need a coherent idempotency strategy. EventBridge archive/replay is useful for recovery, but replaying historical events into current consumers requires schema compatibility and side-effect controls.

## 5. Make processing safe under retries

For a queue consumer, understand visibility timeout, message retention, receive count, batch size, long polling, dead-letter queues, and redrive policies. Visibility hides work temporarily; it does not delete it. Tune it for processing time and expected retries. AWS gives specific Lambda/SQS timeout guidance, including additional allowance for batching. Enable `ReportBatchItemFailures` when using partial batch responses so successful records need not be retried with failed records. For FIFO processing, preserve ordering when reporting failed and unprocessed messages. See [Lambda SQS configuration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html) and [batch error handling](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

Implement idempotency around a stable business operation, not merely the invocation ID. A useful record contains the key, request fingerprint, processing state, expiry, and stored result. Conditional writes prevent two workers from simultaneously claiming the same operation. Decide what happens if the process crashes while a record is marked in progress. An expiry policy must respect the replay window; removing deduplication history too early makes old work repeatable.

Writing a database row and publishing an event are two separate operations. Learn the transactional outbox pattern: commit the business change and outbox record together, then publish asynchronously and mark progress. Publishers and consumers still handle duplicates. For distributed business steps, use a saga with explicit compensating actions, such as releasing a reservation. A compensation is a business operation, not a magical rollback of every external effect.

Use bounded exponential backoff with jitter. Distinguish transient errors from invalid inputs and permanent authorization failures. Retries at several layers multiply load; document the retry owner and total retry budget. A DLQ is an operational work queue with alarms and a runbook, not a place to forget messages.

## 6. Orchestrate multi-step work

Learn Step Functions states, branching, parallel execution, Map, retries, catches, timeouts, service integrations, and execution history. Prefer direct service integrations when a function would only forward a request. Keep large data outside workflow state and pass references.

Standard workflows provide durable orchestration and an exactly-once workflow execution model unless configured retry behavior reruns tasks. Asynchronous Express is at least once; synchronous Express is at most once. These workflow guarantees do not turn all downstream systems into one atomic transaction. Retried activities and external API ambiguity still require careful side-effect handling. Standard and Express also differ in supported patterns, history, duration, and billing. Review [workflow type and execution semantics](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html).

For callback tasks, protect task tokens as credentials. Design cancellation and expiration behavior. For a large Map operation, choose concurrency based on downstream capacity and include a partial-failure policy. Learn when choreography through events simplifies ownership and when orchestration makes business progress easier to understand.

## 7. Lab: resilient order processing

**Build:** API Gateway → Lambda intake → DynamoDB order/outbox records → publisher → SQS → Lambda worker → order status. Use synthetic orders. Provision through SAM, CDK, CloudFormation, or your chosen IaC tool.

1. Define `POST /orders` and `GET /orders/{id}`. Scope reads to the caller's tenant. Require a client-generated idempotency key for creation.
2. Create a table with access patterns for order lookup and operation deduplication. Use a transaction or another documented atomic design for order plus outbox creation.
3. Create a source queue and DLQ, encrypted storage, narrow queue permissions, bounded retries, and explicit retention.
4. Implement validation, conditional claim, state transitions, and duplicate-safe completion. Configure partial batch failure reporting and a concurrency cap.
5. Add alarms for oldest message age, DLQ depth, Lambda errors/throttles, and API failures. Publish an order-completion business metric.
6. Add a publisher with recovery for unpublished outbox records. Document the possibility of a publish succeeding before its acknowledgment is recorded.

**Verify:** Submit the same order twice and concurrently; expect one business order and a consistent response. Force one record in a mixed batch to fail and confirm successful records remain successful. Force a worker timeout after a state change and confirm retry does not repeat the business operation. Break a permission deliberately in the sandbox and trace the failure to the correct boundary. Repair it, redrive a known DLQ message, and reconcile accepted orders against completed plus pending plus failed orders.

**Evidence:** Save the architecture, event schema, sanitized sample logs, retry matrix, alarm screenshots, reconciliation query, and a short incident runbook. Include what your design cannot guarantee.

**Cost:** Estimate requests, compute duration/memory, queue operations, database capacity, logs, traces, and any VPC endpoints or NAT. Run a bounded small batch; avoid an uncontrolled feedback loop between events and object writes.

**Cleanup:** Stop schedules and publishers first, drain or explicitly discard synthetic work, remove event-source mappings, then delete the application stack. Check retained tables, DLQs, object versions, log groups, secrets, and network resources. Do not delete shared infrastructure.

## 8. Extend the lab

Add a Step Functions inventory/payment simulation using a fake payment adapter. Introduce a failure after inventory reservation, execute compensation, and prove the final order state is explainable. Add an EventBridge notification event with two independent subscriber queues. Replay a small archived batch with side effects disabled or safely deduplicated. Compare a synchronous API to an asynchronous accepted-job API using latency and completion metrics.

## Common pitfalls

- Returning success before durable acceptance without explaining the data-loss window.
- Assuming increasing Lambda concurrency always improves throughput.
- Treating timeouts, queue retries, and workflow retries as independent settings.
- Using a single FIFO message group for unrelated work and accidentally serializing everything.
- Giving all functions one broad role or allowing a public function URL without deliberate authentication.
- Logging complete events containing tokens, personal data, or large documents.
- Disabling poison-message retries without preserving a recoverable failure record.

## Mastery checklist

- [ ] I can trace one request through authentication, durable acceptance, processing, and completion.
- [ ] I can explain at-least-once delivery and implement duplicate-safe business effects.
- [ ] I can choose a queue, topic, event bus, stream, or workflow using requirements.
- [ ] I have tested a timeout after a side effect and a partial batch failure.
- [ ] I can set a concurrency budget from downstream capacity.
- [ ] I can recover a DLQ and reconcile records after replay.
- [ ] I can deploy, observe, and clean up the entire application from code.

## Official references

- [AWS Serverless developer guide](https://docs.aws.amazon.com/serverless/latest/devguide/welcome.html) — architecture and implementation entry point.
- [Lambda best practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) — runtime, configuration, and operational considerations.
- [API Gateway API type comparison](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html) — choose by required features.
- [SQS developer guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) — delivery, visibility, security, and DLQs.
- [EventBridge user guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) — event buses, routing, and operational behavior.
- [Powertools for AWS Lambda](https://docs.powertools.aws.dev/) — language-specific utilities and examples.

**Review date:** 2026-10-18. Check current runtime support, quotas, API features, regional availability, and pricing before deploying.
