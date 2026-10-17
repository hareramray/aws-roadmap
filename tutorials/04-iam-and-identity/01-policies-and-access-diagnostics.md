# Tutorial 04.1 — Build and diagnose a prefix-scoped policy

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../04-iam-and-identity.md)

**Mode:** policy exercise with an optional isolated AWS deployment. **Prerequisites:** existing authorized administrator/role-creation access in a sandbox, and a private disposable S3 bucket. **Result:** a policy, test matrix and access-denial diagnosis. Reviewed 2026-10-18.

## 1. Define one job, not a broad job title

The fictional reviewer may list and read objects below `training/` in one lab bucket. They may not upload, delete, read `private/`, or list the whole bucket. Create two tiny dummy objects with an existing authorized setup identity: `training/readme.txt` containing `public-to-reviewer`, and `private/readme.txt` containing `private-dummy`. Keep Block Public Access enabled and use S3-managed encryption initially.

Record the bucket ARN and both object ARNs. General purpose bucket names are part of their ARN; object permissions need the key suffix. Do not substitute an HTTP URL for an ARN.

## 2. Author the policy

Save the following as `reader-policy.json` in UTF-8, replacing the bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTraining",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::REPLACE_WITH_LAB_BUCKET",
      "Condition": {"StringLike": {"s3:prefix": "training/*"}}
    },
    {
      "Sid": "ReadTraining",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::REPLACE_WITH_LAB_BUCKET/training/*"
    }
  ]
}
```

Explain every element aloud. `Version` selects policy language; `Sid` labels a statement; `Effect` allows; `Action` identifies API permissions; `Resource` scopes them; `Condition` restricts the listing request's prefix. `training/*` permits `training/` and deeper prefixes, while a bare `training` request could expose similarly named prefixes and is intentionally absent.

Some service operations support resource-level permissions and others need `*`. Check the service authorization reference before copying this ARN pattern to another service. The policy omits a `Principal` because it is an identity policy attached to a role.

## 3. Validate before attaching

Parse locally:

```powershell
$policy = Get-Content -LiteralPath './reader-policy.json' -Raw -Encoding UTF8 | ConvertFrom-Json
$policy.Statement.Count
```

Expected result: `2`. Parsing proves JSON syntax only, not AWS semantics. In the IAM policy editor, run its policy validation and inspect Access Analyzer findings. If using the CLI with an existing profile, `accessanalyzer validate-policy` accepts a policy document and policy type; review current command syntax before invoking it.

Create a dedicated role whose trust policy allows only the designated sandbox source role to assume it. Attach this policy and no unrelated read-all policy. Use your existing federation/assume-role workflow to obtain the test session; the next tutorial explains trust separately. Do not edit the role through which you administer the account.

## 4. Execute positive and negative cases

Configure the assumed-role profile as `labreader` through your approved workflow, then set the actual bucket name:

```powershell
$labRegion = 'ap-south-1'
$labBucket = 'REPLACE_WITH_LAB_BUCKET'
aws sts get-caller-identity --profile labreader --region $labRegion
aws s3api list-objects-v2 --bucket $labBucket --prefix training/ --profile labreader --region $labRegion
aws s3api get-object --bucket $labBucket --key training/readme.txt --profile labreader --region $labRegion './downloaded-training.txt'
aws s3api get-object --bucket $labBucket --key private/readme.txt --profile labreader --region $labRegion './must-not-download.txt'
aws s3api list-objects-v2 --bucket $labBucket --profile labreader --region $labRegion
```

The training list/read should succeed. The private read and unscoped list should fail with access denial. Test an attempted upload of another dummy file through the same role; it should fail because no `PutObject` allow exists. Do not remove security controls merely to make a console screen load: the console may request discovery permissions not needed by the direct API.

## 5. Introduce limiting-policy scenarios

Use local reasoning or the policy simulator, not account-wide live guardrails. Add a hypothetical identity allow for `PutObject`, then a boundary that allows only reads. The boundary does not grant reads by itself and it limits the identity grant. Add an explicit deny for the training read: that deny overrides an applicable allow.

Now consider SCPs, RCPs, a session policy and an endpoint policy. Record which acts as an organizational/resource/session/network-path constraint. Do not treat them all as interchangeable grant policies. Same-account resource grants to users, roles and assumed-role session ARNs have subtleties; consult the policy-evaluation reference rather than applying one union/intersection diagram universally.

Cross-account access needs appropriate authorization on both sides under the service's rules. A bucket policy cannot repair an explicit organization deny. A KMS-encrypted object can require key authorization even when its S3 object policy permits the read.

## 6. Diagnose one intentional failure

Use the private-object denial. Record the caller ARN, account, Region, action, object ARN, timestamp, request ID and error. Compare the intended role with `get-caller-identity`. Check credentials before policies: expired sessions and missing session tokens are authentication problems, not missing S3 permissions.

Then inspect applicable identity/resource/organization/endpoint/key controls. Where configured, correlate CloudTrail evidence. S3 object reads are data events, so ordinary management event history does not guarantee this read is present. A simulator result is useful but does not replace the real negative test.

**Troubleshooting:** a positive case failing often means a typo in the bucket/key ARN, wrong assumed role, prefix mismatch, explicit deny, or unexpected SSE-KMS key. An unexpectedly successful negative case often means another policy grants access or the request used your administrator profile.

## 7. Review and remove

Document which permission was necessary and which tests prove absence of broader access. Managed policies may change and can be broader than needed; inspect contents rather than trusting a friendly name. Repeat representative tasks before removing apparently unused permissions, so infrequent recovery actions are not silently broken.

**Cleanup/cost:** switch back to the setup identity, delete dummy object versions where applicable and the dedicated bucket, detach/delete the test policy and role, and remove the temporary profile. Never delete shared roles. S3 requests/storage and optional analysis or logging can incur charges; no cost-free deployment is promised.

Sources: [policy evaluation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html), [service authorization reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference.html), [Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html).
