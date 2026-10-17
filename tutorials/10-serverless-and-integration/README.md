# Serverless and integration tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../10-serverless-and-integration.md)

Work through these in order. The first tutorial makes duplicate delivery harmless for one deliberately narrow business operation. The second connects that worker to a workflow and designs the API, identity, and event boundaries around it. Local exercises are free of AWS charges; deployment sections are optional and billable.

| Roadmap topics | Worked coverage and evidence |
| --- | --- |
| Synchronous, asynchronous, and poll based execution; retries and acceptance versus completion | [Idempotent event worker](01-idempotent-event-worker.md): replay identical events, reject conflicting keys, and inspect batch acknowledgments. [API and workflow](02-api-and-workflow-integration.md): follow a request through acceptance, enqueue, and business result. |
| Lambda handler/runtime, environment reuse, deployment packages, IAM, configuration, timeouts, memory, concurrency, logs, versions, aliases | Tutorial 1 supplies a complete handler and SAM stack, bounded concurrency, structured outcomes, retention, and deployment checks. Version/alias release exercise continues in [delivery and recovery](../13-devops-and-delivery/02-deployment-and-recovery.md). |
| SQS standard/FIFO, visibility, retention, DLQ, redrive, partial batch response, idempotency | Tutorial 1 creates a standard queue and DLQ, stores a conditional business result, injects a poison record, and explains the exact FIFO change. |
| API Gateway HTTP/REST/WebSocket, OpenAPI contract, authentication, authorization, Cognito/JWT, IAM, authorizers, CORS, throttling | Tutorial 2 defines and tests a request contract, implements a workflow adapter, and applies an authenticated deployment checklist. It includes explicit adaptations for each API choice. |
| Step Functions Standard/Express, Choice/Task/Retry/Catch, execution history, timeouts, compensation, callback tokens | Tutorial 2 generates a complete Standard definition, starts and inspects executions, and exercises validation failure. It extends the definition with retry, catch, and compensation reasoning. |
| SNS, EventBridge buses, rules, archive/replay, Pipes, Scheduler, versioned event contracts | Tutorial 2 includes a routing pattern, concrete filters, and a delivery/replay experiment for each integration style. |
| Kinesis/MSK retained logs, Amazon MQ protocol migration | Tutorial 2 traces how acknowledgment, replay, ordering, and consumer ownership change; build the retained-stream exercise in [streaming and warehouse design](../14-data-engineering-and-analytics/02-streaming-and-warehouse-design.md). |
| AppSync, resolvers, GraphQL authorization, subscription fan-out | Tutorial 2 maps its status contract to a resolver and tests ownership enforcement before subscription delivery. |
| VPC access, NAT/endpoints, database connections/RDS Proxy, secrets, Powertools, tracing | Tutorial 1's operational extension specifies a connection budget, outbound dependency test, secret boundary, and correlation fields. Existing networking and database tutorials supply the shared infrastructure. |
| Failure recovery, testing, cost, cleanup, operational readiness | Both tutorials require expected results, controlled failures, explicit account/Region checks, a resource ledger, and teardown. |

Prerequisites are Python, PowerShell, and an understanding of IAM. For AWS deployment, also complete [account setup](../../02-account-and-tooling.md), install the tools using their current official instructions, and use a disposable account with a spending alert. Do not copy an IAM policy into a production account merely to make a lab pass.

The queue worker intentionally records an accepted order as its only business effect. Sending an email or charging a payment after the conditional database write introduces a second consistency boundary. Use an outbox/transactional boundary and downstream idempotency before extending the guarantee to external effects.
