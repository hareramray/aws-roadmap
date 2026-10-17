# IAM and identity tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../04-iam-and-identity.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — Build and test least-privilege policies](01-policies-and-access-diagnostics.md) | Prefix-scoped S3 policy and allowed/denied evidence |
| [02 — Federation, cross-account trust and guardrails](02-federation-and-cross-account-trust.md) | Workforce/CI federation design with explicit trust tests |
| [03 — Workload and customer identity](03-workload-and-customer-identity.md) | Runtime role, customer authorization and external identity designs |

## Coverage mapping

| Original heading and subtopics | Tutorial |
| --- | --- |
| Identities/sessions: root, MFA, users, groups, roles, STS, service-linked roles and permission sets | 02, 03 |
| Workforce/application identity: Identity Center org/account instances, SCIM, SAML/OIDC, Cognito, workload roles and CI | 02, 03 |
| Policy JSON, ARNs, resource-level actions, prefix policy, KMS dependencies | 01 |
| Evaluation: implicit/explicit deny, identity/resource policies, boundaries, sessions, SCP/RCP, endpoints, trust, PassRole | 01, 02 |
| Conditions/delegation: ABAC, tags, source identity, confused deputy, external ID, OIDC claims, network conditions | 02, 03 |
| Least-privilege workflow, managed policies, Access Analyzer | 01 |
| Troubleshooting: identity, expiration, signature, Region, denies, CloudTrail, simulation | 01, 03 |
| Prefix-scoped S3 lab; CI trust lab; pitfalls/mastery | 01, 02; verification in all |
| Coverage-map extensions: Verified Permissions, Directory Service, Roles Anywhere | 03 |

Use existing approved sandbox identities. These tutorials never require daily root use or an organization-plan change merely to start learning.
