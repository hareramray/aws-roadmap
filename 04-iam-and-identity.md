# 04 — IAM, federation, and workload identity

[Roadmap index](README.md) · Prerequisites: [account and tooling](02-account-and-tooling.md), [cloud foundations](03-cloud-foundations.md)

Reviewed: **2026-10-18**. Identity is the foundation of every AWS service. Complete the policy exercises before giving applications access to real data. All labs use disposable resources and intentionally scoped permissions.

## Outcomes

Explain authentication versus authorization, select human and workload identities, read IAM policy JSON, reason about policy evaluation, establish scoped cross-account access, and diagnose an access denial without replacing it with administrator access.

## 1. Identities and sessions

The **root user** is the account's exceptional administrative identity. Protect its recovery channels and MFA, do not use it for daily work, and do not create root access keys. Use federated identities and temporary credentials where possible; require MFA for human access and prefer phishing-resistant methods when available. [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

Distinguish these objects:

| Object | Purpose | Important boundary |
| --- | --- | --- |
| IAM user | Long-lived identity for exceptional compatibility needs | Access keys persist until changed or removed |
| IAM group | Assign permissions to several IAM users | Not an assumable identity and not a resource-policy principal |
| IAM role | Identity assumed for a session | Has a trust policy and permissions; does not belong permanently to one person |
| STS session | Temporary credentials for an assumed or federated identity | Has expiration, session context, and possibly narrower permissions |
| Identity Center permission set | Define workforce access provisioned to accounts | Managed through Identity Center rather than hand-editing generated roles |
| Service-linked role | Role tightly integrated with a particular AWS service | Creation, permissions, and deletion have service-specific rules |

Temporary credentials consist of an access key ID, secret access key, and session token. Expiration is useful but not a complete incident-response strategy. Understand how to revoke sessions or remove permissions when an identity is compromised.

## 2. Workforce identity and application identity

IAM Identity Center provides centralized workforce access, often using an existing identity provider. Learn users, groups, assignments, permission sets, session duration, MFA, SCIM provisioning, and lifecycle removal. A worker leaving the organization should lose assignments and active access according to an explicit process.

An **organization instance** supports AWS account access and multi-account permissions. An **account instance** supports application assignments within its scope; it is not a replacement for workforce account access. Setting up Organizations can affect a learner's account plan or credits, so follow [module 02](02-account-and-tooling.md) before creating an organization. Existing organization sandboxes can use their assigned roles. [Identity Center instance types](https://docs.aws.amazon.com/singlesignon/latest/userguide/identity-center-instances.html)

For applications, use the hosting environment's workload role: EC2 instance profile, Lambda execution role, ECS task role, or the appropriate EKS workload identity mechanism. A container image or source repository should not contain access keys. CI/CD systems can federate through an OIDC provider with narrowly constrained claims instead of storing AWS keys.

Amazon Cognito addresses customer-facing application identity. User pools handle user authentication and tokens; identity pools can exchange supported identities for AWS credentials. Workforce federation and consumer login have different trust and lifecycle requirements. Authorization inside your product remains your responsibility even when authentication is managed.

## 3. Read a policy as a decision

An IAM policy statement contains an `Effect`, one or more `Action` values, a `Resource` scope, and optional `Condition` entries. Resource policies and role trust policies also identify a `Principal`. The policy language version string identifies policy syntax, not a document revision date.

Learn to read ARNs: partition, service, Region where applicable, account where applicable, and resource identifier. Not every service uses every field, and some API actions require `Resource: "*"` because they do not support resource-level authorization. Consult the service authorization reference rather than guessing ARN formats.

The following is an **illustrative identity policy**, not a complete deployment. Replace the bucket name with your disposable lab bucket. It grants listing of one prefix and reads of objects inside that prefix:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTrainingPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::REPLACE_WITH_LAB_BUCKET",
      "Condition": {"StringLike": {"s3:prefix": ["training/*"]}}
    },
    {
      "Sid": "ReadTrainingObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::REPLACE_WITH_LAB_BUCKET/training/*"
    }
  ]
}
```

`ListBucket` uses the bucket ARN; `GetObject` uses object ARNs. The policy does not grant upload, delete, reading other prefixes, or KMS decryption. An SSE-KMS object may require additional key permission. A console workflow may request extra discovery APIs; a failed console page does not prove the intended direct API is denied.

## 4. Policy evaluation and guardrails

Begin with implicit denial. Find applicable allows, then check explicit denies and limiting policies. Identity and resource policies interact; permissions boundaries, session policies, service control policies (SCPs), resource control policies (RCPs), and endpoint policies can further constrain access. A boundary or SCP limits permissions; it does not itself grant the workload permission to act. [IAM policy evaluation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

Avoid a simplistic universal “union of all policies” rule. Same-account resource grants to a role ARN, a user ARN, or an assumed-role session ARN have different interactions with implicit denies. Cross-account access generally requires authorization on both sides, subject to service-specific rules. KMS key policies deserve their own evaluation.

A role's **trust policy** answers who may assume it. Its **permissions policy** answers what the resulting session may do. Allowing a caller to invoke `sts:AssumeRole` is not enough if the trust policy rejects it. Conversely, permission to assume a role does not automatically mean permission to pass it to EC2 or another service.

Study `iam:PassRole`: limit allowed role ARNs and intended services because an overly powerful passed role can create privilege escalation. Also inspect permissions to edit policies, attach policies, change trust, create access keys, or alter authorization-relevant tags.

## 5. Conditions and delegated access

Learn request context, string and ARN operators, tag conditions, organization restrictions, secure transport requirements, source identity, and source account/source ARN protections against confused-deputy scenarios. Conditions are service-dependent; an unsupported condition key can produce unexpected results.

Attribute-based access control (ABAC) uses principal and resource attributes to scale permissions. It works only when tag creation and modification are controlled. A user who can set their own privileged department tag may bypass the intended rule.

For third-party cross-account roles, understand external IDs and tenant-specific trust. An external ID is a confused-deputy control, not a password to publish casually. For CI OIDC, restrict issuer, audience, subject, repository, branch or environment, and allowed session behavior according to the provider's claim format.

Identity restrictions based on source IP or VPC endpoints can affect service-to-service requests differently from direct user calls. Test the actual access path, including delegated AWS services, before enforcing a broad deny.

## 6. Least-privilege workflow

1. Write the exact task and API operations the role needs.
2. Identify resource types, dependent actions, and condition keys in the service authorization reference.
3. Start with scoped permissions in a sandbox; use separate deployment and runtime roles.
4. Validate policy syntax and Access Analyzer findings.
5. Exercise allowed and denied cases with the actual session.
6. Review CloudTrail evidence and remove unused permissions after representative usage.
7. Add review ownership, expiration where appropriate, and change history.

Managed policies can accelerate a lab but may be broader than the workload needs and can change. Record policy names, inspect their contents, and move toward a deliberately reviewed permission set. Do not “fix” a denied action by attaching `AdministratorAccess` to an application role.

## 7. Troubleshooting access

Check identity first using `sts get-caller-identity` with the intended profile and Region. Then record the action, resource ARN, request context, time, request ID, and error. Distinguish expired credentials, signature issues, wrong Region, missing allow, explicit deny, resource ownership, endpoint restrictions, and key authorization.

Read the relevant CloudTrail event where recorded. Use policy simulation and Access Analyzer as aids, while recognizing that a simulated allow is not proof that every live service-specific or organization control will permit the request. Never paste credentials into a ticket or commit policy-debug logs containing secrets.

## Lab A — Prefix-scoped S3 access

**Prerequisites:** a temporary sandbox role and the storage basics in [module 07](07-storage-and-backup.md). If live role creation is unavailable, complete the policy and expected-results table locally.

1. In a disposable private bucket, place tiny dummy objects at `training/readme.txt` and `private/readme.txt`. Keep Block Public Access enabled.
2. Create a lab role trusted only by the designated sandbox principal. Attach the illustrative policy after substituting the bucket name.
3. Assume the role through your approved federation workflow. Confirm the account and assumed-role identity.
4. Attempt listing `training/`, reading the training object, reading the private object, and uploading a new object. Use service-managed S3 encryption for this first exercise so KMS permissions do not obscure the lesson.
5. Record expected versus observed results; add a separate policy change only after explaining it.

**Verify:** only the intended list and read succeed. Also test a listing without the prefix to show why context matters. **Cleanup/cost:** remove dummy objects, bucket, role, and inline or customer-managed lab policies after detaching them. S3 requests and stored objects may be billable; IAM configuration does not eliminate those service costs.

## Lab B — Federation and trust review

**Type:** design exercise; no deployment required. Draft a trust policy plan for a CI pipeline deploying one application to a sandbox. Describe the identity provider, accepted claims, deployment role, permitted resources, session duration, and separation from the runtime role. Specify negative tests for a forked repository, unapproved branch, and production account.

**Verify:** a reviewer can identify exactly which workflow gets access and which permission would let it escalate. Explain how access ends when a repository or employee is removed. **Cleanup:** no AWS resources; remove any sample tokens from notes and use fabricated claim values only.

## Pitfalls and mastery

Common failures include missing session tokens, confusing bucket and object ARNs, assuming an SCP grants permissions, unrestricted role trust, broad `PassRole`, credentials in user data, and tag-based access without tag governance.

- [ ] Explain root, user, group, role, session, and permission set.
- [ ] Write separate trust and permission requirements.
- [ ] Predict both positive and negative policy tests.
- [ ] Distinguish a boundary from an allow policy.
- [ ] Use workload roles and controlled federation instead of embedded keys.
- [ ] Diagnose a deny without broadening unrelated access.

Further references: [service authorization reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference.html), [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html), [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html), [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/).
