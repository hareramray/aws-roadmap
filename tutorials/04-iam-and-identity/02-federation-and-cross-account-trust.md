# Tutorial 04.2 — Design federation and test cross-account trust

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../04-iam-and-identity.md)

**Mode:** worked design; optional role assumption only in pre-existing authorized sandboxes. **Result:** workforce and CI trust diagrams with positive/negative cases. Reviewed 2026-10-18.

## 1. Define the actors

Use fictional accounts `111111111111` for nonproduction and `222222222222` for production. A workforce engineer deploys only to nonproduction. A CI workflow builds artifacts and deploys the same application. A third-party auditor may read a narrow report prefix. Replace identifiers only when implementing in actual approved accounts.

Distinguish a person, an IAM user, a group, a role, an STS session and an Identity Center permission set. A group organizes IAM users; it cannot itself assume a role. A role is assumed and produces temporary credentials. A permission set describes access provisioned by Identity Center; manually editing generated roles creates a management conflict.

Root is reserved for exceptional account tasks. Write a root-recovery procedure with MFA and controlled recovery channels, without generating access keys or using root for the exercises. Long-lived IAM users are a compatibility exception, not the default runtime identity.

## 2. Lay out workforce federation

For an existing organization, draw workforce identity provider → Identity Center → assigned group/permission set → account role session. Name `TrainingDevelopers` as the group and define a narrow `TrainingDeploy` permission set. Separate production approval/access from ordinary development.

Record identity source, MFA, session duration, account assignments and deprovisioning. SCIM can synchronize users/groups where configured, while SAML/OIDC federation handles authentication in relevant flows; do not treat provisioning and login as the same event. Remove a departing employee's assignments and follow the organization's session-revocation process.

An Identity Center organization instance supports account access. An account instance supports its application-assignment use cases and is not a substitute for organization workforce account permissions. Do not create an organization just to follow this diagram: account-plan consequences must be checked first. A learner can use an existing approved sandbox role and complete the diagram locally.

## 3. Write the two halves of role assumption

The source identity needs permission for the intended `sts:AssumeRole` operation, and the target role's trust must accept it. The target role's permissions then determine what its session can do.

For the fictional auditor, prepare this **illustrative target trust policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::111111111111:role/ApprovedAuditSource"},
    "Action": "sts:AssumeRole",
    "Condition": {"StringEquals": {"sts:ExternalId": "tenant-training-7f29"}}
  }]
}
```

Attach a separate report-read permissions policy to the target role. The external ID addresses a confused-deputy scenario with the provider's tenant routing; it does not replace precise principal trust, and it is not a universal password. In a real vendor integration, use the vendor's documented unique external-ID workflow.

**Expected tests:** the specified source with the correct external ID can assume the role; a different source cannot; the source with a missing/wrong external ID cannot. An accepted session can read only the report prefix. Assuming the role should not magically grant account administration.

## 4. Design CI OIDC trust

Choose a fictional repository `training/document-portal`, protected branch `main`, and deployment environment `sandbox`. Inventory the CI provider's actual token issuer, audience and subject-claim formats using its official documentation. Do not fabricate a claim syntax and deploy it blindly.

Create a trust requirements table: accept only that issuer, intended AWS audience and the specific approved repository/environment subject. Reject a fork, a pull-request token, another repository and the production environment unless intentionally authorized. The CI job exchanges its token for a bounded role session rather than reading stored AWS access keys.

The deployment role may update only the portal's resources. The runtime role reads only application data. A successful pipeline has no reason to inherit the runtime's database administrator privileges. Protect repository settings and deployment environments because they now influence access to AWS.

## 5. Add conditions without creating bypasses

Try an ABAC design: principal tag `Project=Portal` permits operations on resources tagged `Project=Portal`. Then ask who can modify either tag. If a developer can retag a sensitive resource or assign themselves privileged session tags, the intended boundary may fail.

Define tag-write permissions separately and test missing, mismatched and forged tags. Record source identity/session tags in audit evidence where supported. Use service-specific condition keys; a key supported by S3 is not automatically meaningful to every service.

For AWS service principals accessing your resource, examine source-account/source-ARN conditions where supported to address confused-deputy risks. For source-IP or VPC-endpoint restrictions, test service-to-service requests as well as direct users; broad network denies can reject legitimate delegated calls.

## 6. Bound delegation and organization controls

Consider a developer who can create an EC2 instance and pass any role. They may be able to launch code with stronger permissions than their own. Scope `iam:PassRole` to the intended runtime role and service, and control permissions to edit policies, trust and tags.

Use a local policy table to show an SCP prohibiting an action in nonproduction, an RCP restricting supported resource access, a permissions boundary limiting created roles, and a session policy narrowing one session. These are guardrails, not automatic allows. Do not test a broad deny in a management account.

Service-linked roles have service-specific management behavior. Before deleting one, understand whether the service still needs it and what resources depend on it. This differs from deleting a disposable custom lab role.

## 7. Validate the complete lifecycle

Write acceptance evidence for login, session expiration, denied production access, allowed sandbox deployment and employee removal. If using live sandbox roles, verify account/session identity before each test and retain request IDs rather than credentials.

**Troubleshooting:** assumption failure can come from the source allow, target trust, external ID/claims, organization guardrail or session duration constraints. “The permissions policy is correct” is not enough when trust rejects entry. If a removed assignment appears active, distinguish future login from already-issued sessions and follow documented revocation controls.

**Cleanup/cost:** the design-only path has no AWS resources. For optional implementation, remove only dedicated lab assignments, roles, policies and OIDC providers that have no other consumers. Restore no broad access as a cleanup shortcut. Do not delete existing identity infrastructure.

Sources: [Identity Center instances](https://docs.aws.amazon.com/singlesignon/latest/userguide/identity-center-instances.html), [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html), [OIDC role setup](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html), [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html).
