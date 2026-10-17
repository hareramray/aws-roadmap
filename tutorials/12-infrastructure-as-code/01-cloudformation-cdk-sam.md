# Review and execute CloudFormation, CDK, and SAM changes

[IaC tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../12-infrastructure-as-code.md)

**Outcome:** Create an encrypted SQS queue through a reviewed change set, update a property, detect drift, and compare CDK/SAM-generated templates. Editing files is local. CloudFormation API calls access your account; creating resources and sending messages can incur charges. Use a disposable account/Region and a unique stack name. No credentials belong in a template.

## 1. Write the desired state

Create `queue.yaml` in a new directory:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Disposable IaC queue exercise
Parameters:
  VisibilitySeconds:
    Type: Number
    Default: 60
    MinValue: 30
    MaxValue: 300
Resources:
  Orders:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: !Ref VisibilitySeconds
      MessageRetentionPeriod: 345600
      SqsManagedSseEnabled: true
      Tags:
        - Key: Purpose
          Value: roadmap-iac
Outputs:
  QueueUrl:
    Value: !Ref Orders
  QueueArn:
    Value: !GetAtt Orders.Arn
```

The logical ID `Orders` identifies a resource within a stack. Its physical name is generated, avoiding a globally reused fixed name. `Ref` returns this resource's queue URL; `GetAtt` selects its ARN. These return values differ by resource type. References also express dependencies so CloudFormation can order operations.

The queue uses a parameter with bounded values and an explicit encryption setting. Encryption does not replace an access policy. Avoid adding `DeletionPolicy: Retain` reflexively: retaining a queue intentionally leaves it behind and requires separate ownership/cleanup. For valuable data, choose a resource-supported retention/snapshot policy and a tested restore procedure.

## 2. Prepare commands with failure checks

Set `$env:AWS_PROFILE`, `$env:AWS_REGION`, and `$env:LAB_STACK` to a new sandbox stack name. Use this helper for the remaining commands. It treats native failure as an exception, including Windows PowerShell where `$ErrorActionPreference` alone does not reliably convert native nonzero exit codes into terminating errors.

```powershell
function Invoke-LabAws {
    & aws @args
    if ($LASTEXITCODE -ne 0) { throw "AWS CLI failed: $($args[0]) $($args[1])" }
}
Invoke-LabAws sts get-caller-identity
Invoke-LabAws cloudformation validate-template --template-body file://queue.yaml
```

Inspect the account and Region before proceeding. Template validation checks syntax/structure, not quota, every permission, future success, or application correctness. A stack execution role, if used, needs only the intended resource operations; the caller separately needs the appropriate CloudFormation and pass-role permissions.

## 3. Create a change set, inspect it, then execute

```powershell
Invoke-LabAws cloudformation create-change-set --stack-name $env:LAB_STACK --change-set-name initial --change-set-type CREATE --template-body file://queue.yaml
Invoke-LabAws cloudformation wait change-set-create-complete --stack-name $env:LAB_STACK --change-set-name initial
Invoke-LabAws cloudformation describe-change-set --stack-name $env:LAB_STACK --change-set-name initial
```

Read the resource additions, parameters, and status. The expected change is one queue addition. Only after review, execute:

```powershell
Invoke-LabAws cloudformation execute-change-set --stack-name $env:LAB_STACK --change-set-name initial
Invoke-LabAws cloudformation wait stack-create-complete --stack-name $env:LAB_STACK
Invoke-LabAws cloudformation describe-stacks --stack-name $env:LAB_STACK --query 'Stacks[0].Outputs'
```

Record the queue URL and ARN in a lab inventory. Read its attributes with `sqs get-queue-attributes --queue-url <recorded-url> --attribute-names VisibilityTimeout SqsManagedSseEnabled`. Expect 60 and true. A CREATE change set can leave a review-in-progress stack if never executed; delete that unused stack during cleanup too.

## 4. Preview an update and detect controlled drift

Create `parameters.json` containing `[{"ParameterKey":"VisibilitySeconds","ParameterValue":"90"}]`. Create a new change set named `visibility90` with type `UPDATE`, the same template, and `--parameters file://parameters.json`. Wait, describe, and confirm the queue changes without replacement before executing. Wait for `stack-update-complete`, then verify the queue attribute is 90.

For a controlled drift exercise, use the console or `set-queue-attributes` to change only this lab queue's visibility to 100. Save a UTF-8 JSON file `{"VisibilityTimeout":"100"}` and supply it using `--attributes file://attributes.json`. This deliberately bypasses desired state. Run `detect-stack-drift`, save its detection ID, poll `describe-stack-drift-detection-status` until complete, and inspect `describe-stack-resource-drifts`. Expect a modified visibility property. Drift detection is not universal for every property/resource.

A no-op stack update does not necessarily repair an out-of-band difference. Explicitly create a reviewed template/parameter change, for example to 91, execute it, verify it, then change back to 90 if required. Never interpret a successful no-op response as proof of reconciliation.

Try a parameter value below the minimum in a new change set. Expect rejection before resource update. For a real stack failure, inspect stack events to identify the first causal failure rather than only the final rollback status. Preserve diagnostics, repair the cause, and use the documented rollback/recovery operation for that actual status. Skipping a resource during rollback can leave template and infrastructure inconsistent.

## 5. Compare a CDK application

In a separate already initialized CDK TypeScript project with reviewed pinned dependencies and lockfile, use this application. Its stack must have a different name; it creates a different queue.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as sqs from 'aws-cdk-lib/aws-sqs';

const app = new cdk.App();
const stack = new cdk.Stack(app, 'RoadmapCdkQueue', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
const queue = new sqs.Queue(stack, 'Orders', {
  visibilityTimeout: cdk.Duration.seconds(90),
  retentionPeriod: cdk.Duration.days(4),
  encryption: sqs.QueueEncryption.SQS_MANAGED,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
new cdk.CfnOutput(stack, 'QueueUrl', { value: queue.queueUrl });
```

Run the project's pinned CLI via its package script to synthesize, then inspect the `AWS::SQS::Queue` in `cdk.out`. Context lookups can access AWS even when a command appears to be synthesis; this example has no explicit lookup. Treat context files as reviewed environment-specific inputs. Before optional deployment, inspect `cdk diff`, bootstrap requirements, trusted roles, asset buckets, and generated permissions. Bootstrapping creates persistent infrastructure and is not a free local compiler action. After deployment, verify the same attributes as before and destroy this separate stack when finished.

## 6. Compare a SAM transform

Save this in a third directory as `template.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  Hello:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.13
      Handler: index.handler
      Timeout: 3
      InlineCode: |
        def handler(event, context):
            return {"message": "hello", "value": event.get("value")}
```

Use `sam validate --lint`; if optionally deploying, choose a third unique stack, inspect IAM changes, and invoke the resulting Lambda with one synthetic JSON payload. Inspect both the CLI exit code and Lambda `FunctionError`/payload: successful transport can still contain a handler failure. SAM local execution may require Docker and does not emulate all IAM, networking, quotas, or managed integration behavior. Continue with the [complete SAM queue worker](../10-serverless-and-integration/01-idempotent-event-worker.md) for actual integration testing.

## 7. Ownership, recovery, and cleanup

Before an import exercise, create a separate disposable queue, write matching desired properties with a suitable `DeletionPolicy`, confirm the resource supports import, and generate an IMPORT change set mapping its actual identifier to a new logical ID. Review import restrictions and run drift detection afterward. Import associates ownership; it does not automatically rewrite the resource to match your template. Never import the same queue into two stacks/tools.

Use conditions for intentional optional resources and nested stacks for coherent reuse; cross-stack exports introduce update/delete coupling. StackSets add account/Region targeting and delegated permissions and should first run against one sandbox target. Stack policies protect selected update actions, while termination protection guards stack deletion; neither is a backup of the underlying data.

Delete all three lab stacks through their owning tools and wait for completion. Verify retained resources, logs, SAM/CDK assets, and bootstrap resources separately. Do not remove a shared bootstrap stack. Record the final queue inventory and actual remaining storage. Explain why `Retain`, termination protection, drift detection, and backups solve different problems.

Official references: [change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html), [drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html), [resource import](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html), [CDK practices](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html), [SAM concepts](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html).
