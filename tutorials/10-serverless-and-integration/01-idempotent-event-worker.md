# Build and replay an idempotent event worker

[Serverless tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../10-serverless-and-integration.md)

**Outcome:** Process a batch containing valid, duplicate, conflicting, and malformed orders. A duplicate creates one durable order result; unsuccessful records remain eligible for retry. The local exercise uses an in-memory fake for deterministic tests. The optional AWS exercise uses Lambda, SQS, DynamoDB, and a DLQ and incurs charges.

## 1. Define the business effect precisely

An order contains `order_id` and positive integer `quantity`. Creating an immutable accepted-order row is the entire business effect. The ID is an idempotency key. Reusing an ID with different data is a conflict, not a successful duplicate. This design does not charge a card or send email, so it cannot claim those external effects happen exactly once.

Create a new disposable directory containing `src/worker.py` and `template.yaml`. Use Python for the code below. The package is dependency-free locally; the Lambda runtime supplies Boto3, although production builds should pin/package their SDK dependencies for reproducibility.

```python
import hashlib
import json
import os
import re


def normalize(body):
    value = json.loads(body)
    if not isinstance(value, dict) or set(value) != {"order_id", "quantity"}:
        raise ValueError("invalid fields")
    if not isinstance(value["order_id"], str) or not re.fullmatch(r"[a-z0-9-]{1,40}", value["order_id"]):
        raise ValueError("invalid order_id")
    if type(value["quantity"]) is not int or not 1 <= value["quantity"] <= 100:
        raise ValueError("invalid quantity")
    canonical = json.dumps(value, sort_keys=True, separators=(",", ":"))
    value["fingerprint"] = hashlib.sha256(canonical.encode()).hexdigest()
    return value


def write_aws(value):
    import boto3
    from botocore.exceptions import ClientError
    table = boto3.resource("dynamodb").Table(os.environ["RESULTS_TABLE"])
    try:
        table.put_item(Item={**value, "status": "accepted"},
                       ConditionExpression="attribute_not_exists(order_id)")
        return "created"
    except ClientError as error:
        if error.response["Error"]["Code"] != "ConditionalCheckFailedException":
            raise
        previous = table.get_item(Key={"order_id": value["order_id"]},
                                  ConsistentRead=True).get("Item", {})
        if previous.get("fingerprint") != value["fingerprint"]:
            raise ValueError("idempotency conflict")
        return "duplicate"


def process(event, write):
    failures = []
    for record in event["Records"]:
        try:
            value = normalize(record["body"])
            outcome = write(value)
            print(json.dumps({"message_id": record["messageId"],
                              "order_id": value["order_id"], "outcome": outcome}))
        except Exception as error:
            failures.append({"itemIdentifier": record["messageId"]})
            print(json.dumps({"message_id": record["messageId"],
                              "outcome": "failed", "error_type": type(error).__name__}))
    return {"batchItemFailures": failures}


def handler(event, context):
    return process(event, write_aws)


if __name__ == "__main__":
    results = {}
    def write_local(value):
        previous = results.get(value["order_id"])
        if previous and previous["fingerprint"] != value["fingerprint"]:
            raise ValueError("idempotency conflict")
        results[value["order_id"]] = value
        return "duplicate" if previous else "created"
    bodies = ['{"order_id":"a","quantity":2}',
              '{"order_id":"a","quantity":2}',
              '{"order_id":"a","quantity":3}', '{"order_id":"b","quantity":0}']
    event = {"Records": [{"messageId": str(i), "body": body}
                          for i, body in enumerate(bodies)]}
    response = process(event, write_local)
    assert response == {"batchItemFailures": [{"itemIdentifier": "2"}, {"itemIdentifier": "3"}]}
    assert len(results) == 1
    print("PASS: one effect, duplicate accepted, two failures")
```

Run `python .\src\worker.py`. Expect `created`, `duplicate`, two failures, and the final PASS line. The fake verifies business logic, not distributed concurrency. DynamoDB's conditional write supplies the atomic check in AWS. A timeout after a successful write is safe to retry because the next attempt compares the persisted fingerprint.

## 2. Describe the optional AWS resources

Save this SAM template. Review runtime support before deployment. The queue visibility of 90 seconds exceeds six times this function's 10-second timeout; no batching window is configured. A DLQ receive count is a lab choice, not a universal recommendation.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  DeadLetters:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600
      SqsManagedSseEnabled: true
  WorkQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 90
      SqsManagedSseEnabled: true
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt DeadLetters.Arn
        maxReceiveCount: 3
  Results:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: order_id
          AttributeType: S
      KeySchema:
        - AttributeName: order_id
          KeyType: HASH
  Worker:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.13
      Handler: worker.handler
      CodeUri: src/
      Timeout: 10
      MemorySize: 128
      ReservedConcurrentExecutions: 2
      Environment:
        Variables:
          RESULTS_TABLE: !Ref Results
      Policies:
        - SQSPollerPolicy:
            QueueName: !GetAtt WorkQueue.QueueName
        - Statement:
            - Effect: Allow
              Action: [dynamodb:PutItem, dynamodb:GetItem]
              Resource: !GetAtt Results.Arn
      Events:
        Orders:
          Type: SQS
          Properties:
            Queue: !GetAtt WorkQueue.Arn
            BatchSize: 4
            FunctionResponseTypes: [ReportBatchItemFailures]
            ScalingConfig:
              MaximumConcurrency: 2
  WorkerLogs:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/lambda/${Worker}
      RetentionInDays: 7
Outputs:
  QueueUrl:
    Value: !Ref WorkQueue
  QueueArn:
    Value: !GetAtt WorkQueue.Arn
  DeadLetterUrl:
    Value: !Ref DeadLetters
  TableName:
    Value: !Ref Results
```

## 3. Deploy deliberately and record identifiers

Use an established AWS CLI profile and Region. Confirm `aws sts get-caller-identity` matches your sandbox. Estimate Lambda requests/duration, SQS operations, DynamoDB writes/reads, logs, and SAM artifact storage. Verify concurrency quota leaves room for the reservation; a tiny account quota can reject deployment. SAM CLI, Python, AWS CLI, credentials, and permissions are prerequisites, not installed by this document.

```powershell
sam validate --lint
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed' }
sam deploy --guided
if ($LASTEXITCODE -ne 0) { throw 'Deployment failed; inspect stack events' }
```

Choose a unique stack name and record every output plus the generated artifact bucket/prefix. Review the IAM change set. Export output values as `$env:LAB_QUEUE_URL` and `$env:LAB_TABLE`; do not substitute a production queue. `sam deploy` is the billable boundary.

## 4. Send, inspect, and deliberately fail

Write one UTF-8 file without a BOM using PowerShell, then send it twice. Check the native exit code after each AWS command.

```powershell
[IO.File]::WriteAllText((Join-Path $PWD 'order.json'), '{"order_id":"a","quantity":2}', [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $env:LAB_QUEUE_URL --message-body file://order.json
if ($LASTEXITCODE -ne 0) { throw 'First send failed' }
aws sqs send-message --queue-url $env:LAB_QUEUE_URL --message-body file://order.json
if ($LASTEXITCODE -ne 0) { throw 'Duplicate send failed' }
aws dynamodb scan --table-name $env:LAB_TABLE --consistent-read --select COUNT
if ($LASTEXITCODE -ne 0) { throw 'Result read failed' }
```

Allow bounded polling for asynchronous completion; an immediate count of zero is not failure. Expect one row, even though two sends have different SQS message IDs. Inspect function logs and queue oldest-message age. Then change quantity to zero and send once. The message fails, becomes visible again, and eventually reaches the DLQ after repeated receives; exact timing is asynchronous. Do not repeatedly redrive an unfixed poison message. Correct its content or quarantine it with a reason.

For FIFO, use FIFO queues, message groups/deduplication IDs, and stop processing after a failed record in a group, returning failed and unprocessed IDs. This standard-queue loop is intentionally not a complete FIFO ordered processor. For an external payment, replace the single-row effect with an appropriate transactional outbox plus downstream idempotency.

## 5. Operate and clean up

Access denied usually means the execution role lacks the specific table or queue permission; a missing batch response setting causes successful records to replay. Increasing concurrency cannot fix a failing dependency. Reuse SDK clients in a production implementation after measuring initialization; avoid request-specific mutable globals. If adding VPC access, test registry/service/secret dependencies and plan endpoints or NAT. A public subnet does not itself give a Lambda function internet access. Cap database connection demand and evaluate RDS Proxy only for compatible databases.

Add correlation IDs and Powertools/tracing only after deciding which fields may be logged. Measure memory/duration rather than assuming the lowest memory is cheapest. Secrets belong in a managed secret store, not event logs or plaintext environment values.

Stop producers, inspect/save required DLQ evidence, then run `sam delete --stack-name <your-recorded-stack>` with the actual name. Verify the stack, Lambda mapping/function, queues, table, and log group are gone. Check the SAM artifact prefix and any retained deployment resources; remove only objects created for this lab. SQS deletion intentionally discards remaining synthetic messages. No endpoint, NAT gateway, or always-running compute is required by this stack.

Explain: Why does a delivery ID differ from an idempotency key? What happens after a write succeeds but acknowledgment is lost? Why does visibility not guarantee exclusive processing? What additional boundary appears if you add email?

Official references: [Lambda SQS configuration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html), [partial batch failure handling](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html), [SAM SQS event](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-function-sqs.html), [DynamoDB conditional operations](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html).
