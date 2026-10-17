# Connect an API contract, workflow, and event consumers

[Serverless tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../10-serverless-and-integration.md)

**Outcome:** Generate a validated order-processing workflow, trace acceptance separately from completion, and design authenticated API/event integrations. Complete the [worker tutorial](01-idempotent-event-worker.md) first. Local definition generation makes no AWS calls. Creating/executing a state machine, API, scheduler, or event route is optional and billable.

## 1. Establish the contract and identity boundary

The request is `POST /orders` with JSON `{"order_id":"a","quantity":2}`. A successful submission returns HTTP 202 plus an opaque job identifier. A separate authenticated status route returns the stored business outcome. HTTP 202 does not mean the worker completed. The API must derive tenant/owner identity from verified authentication and store it with the request; accepting an arbitrary `owner` field from a client is not authorization.

Keep this tutorial's synthetic `order_id` global within one sandbox. For multiple tenants, make the database key include the trusted tenant ID. The status lookup must check that same identity. A UUID that is hard to guess does not remove the authorization requirement.

Compare API Gateway HTTP APIs for a small JWT/IAM HTTP service, REST APIs when their additional controls are required, and WebSocket APIs for connection-oriented delivery. Check their current feature matrix. An API key is not user authentication. CORS only governs browser cross-origin behavior. Cognito can issue tokens, but the API still needs the correct issuer, audience, claims, and per-record authorization.

## 2. Generate a complete state machine locally

Save this as `make_workflow.py`. It accepts a real queue URL when available, or a nondeployable placeholder. The Standard workflow validates basic fields and enqueues the order. Domain validation remains in the worker. The `Send` result is discarded so the original business input remains the output.

```python
import json
import os
from pathlib import Path

queue = os.environ.get("LAB_QUEUE_URL", "REPLACE_WITH_SANDBOX_QUEUE_URL")
definition = {
    "Comment": "Synthetic order acceptance; worker completion is separate",
    "StartAt": "Validate", "TimeoutSeconds": 60,
    "States": {
        "Validate": {"Type": "Choice", "Choices": [{
            "And": [{"Variable": "$.order_id", "IsPresent": True},
                    {"Variable": "$.order_id", "IsString": True},
                    {"Variable": "$.quantity", "IsPresent": True},
                    {"Variable": "$.quantity", "IsNumeric": True},
                    {"Variable": "$.quantity", "NumericGreaterThan": 0}],
            "Next": "Send"}], "Default": "Invalid"},
        "Send": {"Type": "Task", "Resource": "arn:aws:states:::sqs:sendMessage",
            "Parameters": {"QueueUrl": queue, "MessageBody.$": "States.JsonToString($)"},
            "ResultPath": None,
            "Retry": [{"ErrorEquals": ["States.TaskFailed"], "IntervalSeconds": 2,
                       "MaxAttempts": 2, "BackoffRate": 2}],
            "Catch": [{"ErrorEquals": ["States.ALL"], "ResultPath": "$.delivery_error",
                       "Next": "DeliveryFailed"}], "Next": "Accepted"},
        "Accepted": {"Type": "Succeed"},
        "Invalid": {"Type": "Fail", "Error": "InvalidOrder"},
        "DeliveryFailed": {"Type": "Fail", "Error": "QueueDeliveryFailed"}
    }
}
Path("workflow.json").write_text(json.dumps(definition, indent=2), encoding="utf-8")
Path("valid.json").write_text('{"order_id":"workflow-a","quantity":2}', encoding="utf-8")
Path("invalid.json").write_text('{"order_id":"workflow-b","quantity":0}', encoding="utf-8")
assert definition["States"]["Send"]["Next"] == "Accepted"
assert definition["States"]["Validate"]["Default"] == "Invalid"
print("Wrote workflow.json, valid.json, invalid.json; no AWS calls")
```

Run `python .\make_workflow.py`, inspect all three files, and explain why `Accepted` cannot be named `OrderCompleted`. The ARN uses the commercial AWS partition; use the documented partition-specific integration ARN for other partitions. A broad transient retry here deliberately demonstrates duplicate-safe queue submission, but production should narrow retryable errors and alarm on persistent permission/configuration failures.

## 3. Optional: deploy and observe the workflow

Use the previous stack's queue URL and ARN. Create a dedicated execution role trusted by `states.amazonaws.com`, constrained with source account/ARN where applicable, whose only business action is `sqs:SendMessage` on that queue ARN. Your deployment identity needs tightly scoped `iam:PassRole`. Set `$env:LAB_WORKFLOW_ROLE`, `$env:LAB_WORKFLOW_NAME`, and profile/Region; confirm caller identity before creating anything. This role is separate from the Lambda execution role.

```powershell
python .\make_workflow.py
aws stepfunctions validate-state-machine-definition --definition file://workflow.json --type STANDARD
if ($LASTEXITCODE -ne 0) { throw 'Definition validation request failed' }
# Inspect result=OK and diagnostics before continuing.
aws stepfunctions create-state-machine --name $env:LAB_WORKFLOW_NAME --definition file://workflow.json --role-arn $env:LAB_WORKFLOW_ROLE --type STANDARD
if ($LASTEXITCODE -ne 0) { throw 'Creation failed' }
```

Record the returned ARN as `$env:LAB_WORKFLOW_ARN`. Start a uniquely named execution; use a new name for the invalid input exercise.

```powershell
aws stepfunctions start-execution --state-machine-arn $env:LAB_WORKFLOW_ARN --name ('valid-' + [guid]::NewGuid().ToString('N')) --input file://valid.json
if ($LASTEXITCODE -ne 0) { throw 'Start failed' }
```

Save the execution ARN and use `describe-execution` and `get-execution-history` with that exact ARN. Expect a successful enqueue execution, followed independently by one DynamoDB business row. Start `invalid.json`; expect `InvalidOrder` and no new row. Check queue metrics and Lambda logs as well as workflow history. An API success alone cannot prove processing correctness.

## 4. Build the API adapter and test failure ownership

For an authenticated deployment, an API Lambda adapter should validate the full request with the worker's `normalize` function, derive tenant identity, and call `StartExecution` with a stable job name and canonical business input. That validator adds an internal fingerprint: construct the forwarded payload explicitly as `{"order_id": validated["order_id"], "quantity": validated["quantity"]}`. Do not forward the fingerprint, because the downstream worker accepts exactly those two business fields and computes its own fingerprint. A multi-tenant extension must update both producer and consumer contracts deliberately. Standard execution name/idempotency rules have a bounded retention scope; they do not replace business idempotency in DynamoDB. Restrict its role to the one state-machine ARN. Return only a job ID, not internal stack details or exception traces.

Use a contract test table before exposing the route:

| Request | Required result |
| --- | --- |
| Missing/expired token | Unauthorized; no execution created |
| Valid identity, valid body | Accepted plus job ID; eventually one business row |
| Same business key, identical body | Documented duplicate behavior; no second row |
| Same key, changed quantity | Conflict or later explicit failed status, never silent replacement |
| Wrong tenant asks for status | Denied or indistinguishable not-found response |
| Timeout after accepted submission | Safe client retry using the same key |

A direct API Gateway service integration can eliminate the adapter when its mapping/authorization features meet the contract. AppSync can expose `createOrder` and `orderStatus` with field/resolver authorization; connect the trusted identity to the same ownership key and test cross-tenant subscription delivery. WebSocket connection authentication also needs authorization when delivering later messages; a stale connection must not preserve revoked access indefinitely.

## 5. Extend the event path with observable experiments

Use one versioned envelope: `id`, `type`, `version`, `occurred_at`, `correlation_id`, and `data`. If applying it to the existing worker, explicitly unwrap and validate `data`; sending the envelope unchanged to its two-field validator should fail. This intentional schema check prevents accidental silent acceptance.

For EventBridge, create a rule pattern such as `{"source":["tutorial.orders"],"detail-type":["OrderAccepted"]}` and send one matching and one nonmatching event. Attach an isolated target queue with a resource policy restricted to that rule ARN. Expect only the matching event at the target. Record target delivery failures separately from worker failures. An archive replay should duplicate the matching event, leaving one business result because the key remains stable.

For SNS, subscribe two distinct SQS queues and publish one event. Expect one independent copy per subscriber; the subscribers must not compete on the same queue when both need every event. Configure topic-to-queue permissions with source ARN restrictions and account for the SNS envelope unless raw message delivery is deliberately selected.

For Pipes, connect an isolated source queue to a target with a filter and optional enrichment. Test acknowledgment when the target fails and inspect its source-specific behavior. For Scheduler, use one one-time schedule with a dedicated role and bounded retry policy, record its time zone, and delete the schedule afterward. Do not use recurring schedules for an initial test.

For retained event logs, repeat the [stream replay exercise](../14-data-engineering-and-analytics/02-streaming-and-warehouse-design.md). Kinesis/MSK consumer checkpoints differ from SQS deletion. For Amazon MQ, preserve the application's protocol/acknowledgment contract and test reconnect/redelivery before treating it as a queue drop-in replacement.

## 6. Recovery, economics, and teardown

Standard, asynchronous Express, and synchronous Express have different execution/delivery guarantees; external side effects still need idempotency. Learn callback task tokens using a separate controlled approval workflow; keep tokens secret and bind the responding actor to the business operation. Compensation should reverse a completed business action with a new idempotent action, not assume distributed rollback exists.

Estimate API requests, workflow transitions or Express duration, messages, archives, logs, and downstream calls. Stop executions still running, delete the state machine and any API/integrations, remove schedules, rules/targets, archives, subscriptions, and dedicated roles, then delete the worker stack. Check for retained logs. Deleting a state machine does not retract messages already accepted by SQS.

Explain which component retries each boundary, how a replay changes cost, why a successful state machine does not prove the worker completed, and which identity authorizes a status read.

Official references: [SQS workflow integration](https://docs.aws.amazon.com/step-functions/latest/dg/connect-sqs.html), [workflow types](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html), [definition validation](https://docs.aws.amazon.com/step-functions/latest/apireference/API_ValidateStateMachineDefinition.html), [API Gateway API choices](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html), [EventBridge patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html).
