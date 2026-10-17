# Tutorial 04.3 — Separate workload credentials from customer authorization

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../04-iam-and-identity.md)

**Mode:** worked identity designs plus an optional runtime-role test. **Result:** four trust flows and a customer authorization test table. Reviewed 2026-10-18.

## 1. Define one application and three users

The document portal has customers Alice and Bob and operator Priya. Alice owns document `doc-a`; Bob owns `doc-b`; Priya can inspect operational status but cannot download customers' documents simply because she operates the service.

An application worker needs `GetObject` for original files and `PutObject` for thumbnails. It does not need to administer IAM or read all buckets. Write those permissions against one dedicated bucket/prefix set. Use a distinct deployment identity for creating resources.

This creates two separate questions: may the worker call S3, and may Alice request this particular document? AWS workload authorization and application customer authorization must both be correct.

## 2. Map runtime identity to the hosting model

| Host | Credential mechanism to investigate | Frequent mistake |
| --- | --- | --- |
| EC2 | Instance profile containing a role; SDK credential provider | Keys in an AMI or user data |
| Lambda | Function execution role | Reusing a broad deployment role |
| ECS | Task role for application API calls; execution role for supported task startup work | Giving the application the wrong role or relying on host credentials |
| EKS | Supported workload identity, such as Pod Identity or IRSA according to design | Allowing all pods to inherit node permissions |
| External workload | OIDC federation or IAM Roles Anywhere where suitable | Copying a permanent user key to every machine |

For each row draw how a short-lived credential reaches the SDK, who controls trust, and how expiration is refreshed. The SDK's standard credential chain is useful only if no unexpected environment variable or local credentials override the intended role.

**Worked negative case:** set a hypothetical developer access key in a container environment. Even with a correct task role, the application may use the higher-precedence credential source. Remove embedded/static credentials and verify the resulting caller identity rather than assuming the attached role is always used.

## 3. Optional EC2 runtime proof

Use a disposable instance from the compute tutorial with IMDSv2 required, Session Manager access, and an instance role containing the narrow lab object permissions. Confirm the role trust uses the EC2 service and the instance profile is attached. Do not retrieve or print raw metadata credentials.

Inside that instance, use its installed AWS CLI/SDK with the runtime credential chain. Set an explicit Region for service calls, but do not pass your laptop's human profile. Invoke `sts get-caller-identity` and record only the role/account identity. Read an allowed dummy object, then attempt another prefix; the latter must fail.

If caller identity shows an unexpected principal, inspect credential-source configuration without displaying secrets. If no credentials are found, examine the profile attachment, agent/runtime environment, metadata options and SDK support. IMDSv2 uses tokens and does not justify loosening the policy to make obsolete software work.

For ECS/EKS, reproduce the same allowed/denied test in a dedicated task/pod during their tutorials. Document task/execution/node/workload roles separately. This is an acceptance test specification here, not a claim that a cluster has been created.

## 4. Implement customer authentication as a flow

For a design-only Cognito user-pool exercise, define a browser application client, an allowed callback URL under your controlled development origin, a sign-in flow with PKCE appropriate to the public client, and token validation at the API. A public browser client cannot keep a client secret confidential.

The browser signs in and receives tokens. The API validates issuer, audience/client context, signature, expiry and intended token use using the documented Cognito integration. Decoding a JWT without verifying it is not authentication. Decide whether the API expects an access token or another documented authorization mechanism; do not interchange ID and access tokens casually.

An identity pool is a different component: it can exchange supported identities for temporary AWS credentials. If the browser genuinely needs direct S3 access, constrain that role's data scope. Otherwise, the API can issue a narrow presigned operation after authorization. Neither design permits a client to choose another customer's prefix unvalidated.

## 5. Add an authorization decision

Create this table before writing application code:

| Principal | Action | Resource | Expected |
| --- | --- | --- | --- |
| Alice | download | doc-a | Allow |
| Alice | download | doc-b | Deny |
| Bob | change owner | doc-a | Deny |
| Priya | view processing status | doc-a | Allow under operator policy |
| Priya | download content | doc-a | Deny unless separately authorized |

Implement ownership in a database-backed application check or evaluate an external policy service such as Verified Permissions. For the latter, define principal/action/resource entities and a schema, then model a rule permitting document owners to read their documents. Test the matrix in its policy tooling before integrating authorization calls. Include missing-owner and malformed-context cases; absence of expected data must not become an allow.

Verified Permissions uses application authorization policies; it does not replace IAM permissions for the service's own AWS calls. Keep policy changes reviewed and versioned. Cache decisions only with a clear expiry and revocation strategy, especially when ownership can change.

## 6. Extend identity to an external server

For a fictional on-premises batch processor, design IAM Roles Anywhere using a trusted certificate authority/trust anchor, certificate identity, profile and target role. Draw private-key storage, certificate renewal and revocation. The processor authenticates through supported certificate-based mechanisms to receive temporary AWS credentials.

Test on paper what happens when the certificate expires, the trust anchor is disabled, or the role permissions are removed. A certificate copied to every server recreates the shared-secret problem; use distinct identities and protect private keys. Do not purchase or deploy a CA merely for this first exercise.

For Windows enterprise applications, compare Directory Service choices such as AWS Managed Microsoft AD and AD Connector according to directory hosting versus connection needs. Include DNS, network connectivity, administrator responsibility and user deprovisioning. Simple AD has a separate maintenance notice; do not assume every historical directory option accepts new deployments.

## 7. Finish with access-lifecycle evidence

Produce a diagram separating workforce login, workload credentials, customer login and customer authorization. Add credential expiration, rotation/revocation, audit fields and failure behavior to each arrow. Root credentials and IAM groups should not appear as customer-runtime mechanisms.

**Troubleshooting:** distinguish login failure, token verification failure, application denial and AWS API denial. A customer receiving HTTP 403 may be correctly blocked even while the worker can access S3. Preserve request IDs and sanitized entity identifiers, not tokens.

**Cleanup/cost:** remove only dedicated test instances/roles, app clients, test users, policy stores and external identity objects created for the optional exercise. Identity integrations may have associated service charges; retain nothing merely because it is not compute. Design-only branches create no infrastructure.

Sources: [Cognito documentation](https://docs.aws.amazon.com/cognito/), [Verified Permissions](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html), [Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html), [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
