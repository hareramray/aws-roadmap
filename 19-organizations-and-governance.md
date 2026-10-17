# 19. Organizations and governance

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete [account setup](02-account-and-tooling.md), [identity](04-iam-and-identity.md), [security](09-security-and-compliance.md), [infrastructure as code](12-infrastructure-as-code.md), and [operations](16-observability-and-operations.md).

Learn to design a multi-account operating environment with clear ownership, access boundaries, central evidence, repeatable provisioning, and tested controls. You should be able to explain how Organizations, Control Tower, IAM Identity Center, IAM policies, SCPs, RCPs, and detective controls work together without confusing their responsibilities.

**Beginner spending consideration:** keep this module's first exercise design-only. The current AWS Free Tier FAQ says credit-earning activities expire and eligibility to earn more ends when you join an AWS Organization or set up a Control Tower landing zone. Account experiences and credit terms differ by enrollment date, including an October 2026 signup change. Verify the treatment of already-earned credits, required plan changes, and billable services for your account before an optional live exercise; do not assume every account follows an older tutorial's terms. [AWS Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/).

## 1. Understand why accounts are architectural boundaries

An AWS account provides an important boundary for resource ownership, permissions, billing, and many service quotas. Separate accounts can reduce the impact of mistakes and compromise, clarify ownership, and support different control requirements. They also create operational work: access, inventory, networking, logging, billing, account lifecycle, and support must all be coordinated.

Use business and security requirements to choose boundaries. Common separations include production versus nonproduction, independent workloads, regulated data, shared infrastructure, security tooling, and centralized logs. Avoid both extremes: one account for everything and an account for every tiny component without an operating model.

Keep ordinary workloads out of the management account. Protect management-account access carefully, establish a small set of authorized administrators, and use delegated administration for supported services. The management account has organization-wide responsibilities and different guardrail behavior from member accounts.

Understand that account separation is not automatic network isolation, perfect data isolation, or a substitute for application authorization. Cross-account roles, resource policies, shared networks, and centralized services can intentionally bridge boundaries and must be reviewed.

## 2. Design Organizations and organizational units

Learn the organization root, organizational units (OUs), member accounts, management account, consolidated billing, all-features mode, service integration, delegated administrators, and policy inheritance.

Group accounts by similar governance requirements rather than blindly copying the company reporting structure. A small design might include:

| OU or account group | Purpose | Typical concern |
| --- | --- | --- |
| Security | Log archive and security administration | Evidence integrity and separation of duties |
| Infrastructure | Shared networking and platform services | Broad impact of changes |
| Workloads / Nonproduction | Development and test environments | Rapid iteration within bounded permissions |
| Workloads / Production | Customer-facing systems | Change control, recovery, and access discipline |
| Sandbox | Experiments with limited exposure | Cost, expiration, and external access |
| Policy staging | Testing organizational controls | Preventing fleet-wide mistakes |
| Suspended or quarantine | Account lifecycle or incident containment | Preserve evidence and avoid accidental destruction |

A quarantine design must be tested. Broad denies can also block security responders, log delivery, cleanup, or backup operations. Account movement between OUs changes effective controls; treat it as a production change with a before/after access analysis.

## 3. Learn permission boundaries precisely

Identity and resource policies grant permissions subject to the applicable policy evaluation rules. Organizational authorization policies constrain permissions. An explicit deny overrides an otherwise applicable allow, but the complete evaluation also depends on policy type, principal, resource policy, and request context.

**Service control policies (SCPs)** set maximum permissions for principals in member accounts. They do not grant access, do not constrain the management account, and have documented exceptions such as service-linked roles. A member-account administrator cannot override an applicable SCP deny merely by attaching AdministratorAccess. Read AWS's evaluation and exception documentation before applying a policy. [SCP documentation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html).

**Resource control policies (RCPs)** constrain access to supported resources in member accounts, including access by relevant external principals. They also do not grant permissions, do not cover every service, and have documented exceptions. An RCP and an SCP address different sides of the access boundary. [RCP documentation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html).

Other mechanisms solve different problems:

- **IAM permissions boundary:** limits the permissions an IAM user or role can receive; useful in delegated role creation.
- **Session policy:** restricts a particular temporary-credential session.
- **Resource policy:** controls access to a particular supported resource, including appropriate cross-account grants.
- **Trust policy:** defines who can assume a role and under which conditions.
- **Tag policy:** standardizes tagging rules and supported enforcement behavior; it is not universal authorization or a guarantee every resource has tags.
- **Backup policy:** centrally defines supported backup requirements; actual jobs and restores still need verification.
- **Declarative policy:** configures supported service behavior centrally; inspect its exact scope and inheritance rather than treating it as another IAM allow policy.

Never simplify policy troubleshooting to “the IAM role has AdministratorAccess.” Inspect all relevant identity, resource, organizational, key, endpoint, session, and trust policies.

## 4. Build a landing zone with Control Tower or managed automation

A landing zone is an operating foundation: account structure, access, logging, networking, security baselines, provisioning, and governance. AWS Control Tower helps establish and govern a landing zone, but applications and their configuration remain your responsibility.

Control behavior differs:

| Behavior | What it does | Main learning point |
| --- | --- | --- |
| Preventive | Stops defined disallowed actions or behavior | Verify policy scope and exceptions |
| Detective | Identifies configured noncompliance | Detection does not prevent the initial change |
| Proactive | Checks applicable provisioning requests before creation | Control Tower proactive checks use CloudFormation hooks, so coverage is not every possible API path |

AWS distinguishes control behavior from guidance categories such as mandatory and elective. Check each control's implementation and regional support rather than assuming identical coverage. [Control Tower control behavior](https://docs.aws.amazon.com/controltower/latest/controlreference/control-behavior.html).

Study Account Factory and appropriate account vending automation, including supported Terraform-based workflows where relevant. A request should capture owner, environment, business purpose, cost center, data sensitivity, required Regions, network needs, lifecycle, and support contacts. Provision baseline roles, logging, budgets, security configuration, and approved deployment access automatically.

Version the baseline and monitor drift. A one-time secure setup is insufficient if future accounts miss controls or teams silently replace baseline resources. Account enrollment, OU registration, landing-zone updates, and existing-resource handling need a planned process.

## 5. Centralize identity without losing emergency access

Use federation and IAM Identity Center permission sets for human access where appropriate. Connect workforce identities to the organization's identity lifecycle, enforce suitable authentication, assign access to groups, and keep sessions and privileged roles scoped to actual duties.

Separate production administration, security investigation, billing analysis, network operation, and developer access. Build an access review that checks both human assignments and cross-account machine roles. Temporary credentials still need least privilege and useful audit trails.

Prepare emergency access that survives an identity-provider outage. Document custody, authentication, authorization, alerting, expiration, and review after use. An emergency procedure that nobody can execute during an outage is not operational; a permanent shared administrator password is not an acceptable substitute.

Plan root-user governance and account recovery. Verify current centralized root-management capabilities and prerequisites before adopting them. Keep ownership email addresses, alternate contacts, billing responsibility, and recovery procedures accurate throughout the account lifecycle.

## 6. Centralize evidence, security, and financial accountability

Design organization-wide CloudTrail collection, supported configuration aggregation, security findings, and protected log storage. Separate evidence writers from readers and from retention administrators. Define encryption keys, access patterns, retention, and the recovery path if a workload account is compromised.

Centralize supported GuardDuty, Security Hub, Inspector, Macie, and other security integrations according to the organization's needs. Evaluate Region activation, delegated-admin behavior, newly created accounts, and configuration drift. Aggregating findings does not mean every underlying protection is enabled everywhere.

Use cost allocation tags, account ownership, cost categories, budgets, and billing exports. Decide how shared network, security, observability, support, and commitment charges are allocated. An invoice can be technically correct while being useless for team accountability if shared costs have no owner.

Define a control exception process: business reason, affected resources, compensating controls, owner, approver, expiration, and periodic review. Governance should make the approved path easy and the exceptional path visible; a pile of undocumented permanent exceptions defeats the purpose.

## Practical exercise: design and test a landing-zone policy

**Required design-only exercise:** create an OU diagram, account inventory, permission-set matrix, control register, and account-vending checklist for a small company with development, production, and security teams.

1. Assign every account an owner, purpose, cost center, data classification, and lifecycle state.
2. Select one control, such as preventing a specific destructive logging action in a disposable test account. Explain its purpose and scope.
3. Draft an SCP policy and review it against the official service authorization reference. Document which IAM permissions it constrains and which access it does not grant.
4. Write a test matrix: allowed baseline action, intended denied action, administrator action, security response, and required service behavior.
5. Define rollback from a management/delegated administration path that the proposed control cannot accidentally block.
6. Review external access separately and decide whether resource policies or an RCP are needed for supported resources.

**Optional live exercise:** only use an organization intended for experimentation after accepting its billing and Free Tier implications. Attach the tested control first to a policy-staging OU with one empty test account. Verify actual allowed and denied API calls with the same role and request context. Never first attach an untested broad deny at the organization root.

**Verification:** retain redacted request IDs, denied-action evidence, policy attachments, and a demonstrated rollback. Confirm the policy does not itself grant the previously ungranted action. Use actual evaluation evidence; a successful JSON syntax check alone proves little about effective permissions.

**Cost and cleanup:** design-only work is free of AWS resource charges. Live foundations may create billable Config recording, CloudTrail storage/data events, security services, network resources, and support-related costs. Remove the temporary policy attachment and lab resources using the documented rollback. Do not close accounts or dismantle a shared landing zone as casual cleanup; account closure and organization exit have distinct requirements and consequences.

## Pitfalls and mastery

- Attaching a broad region-deny policy without considering global services and documented exceptions.
- Assuming controls cover the management account or every resource service.
- Giving all teams permanent administrative access because accounts are separate.
- Creating accounts faster than ownership, logging, billing, and security can be maintained.
- Mistaking a compliance dashboard for complete application security.
- Automatically deleting quarantined resources before incident evidence is preserved.

- [ ] I can justify account and OU boundaries.
- [ ] I can explain why SCPs and RCPs do not grant permissions.
- [ ] I can distinguish preventive, detective, and proactive control coverage.
- [ ] I can trace a denied request through the relevant policy layers.
- [ ] I can describe account provisioning, access review, emergency access, and retirement.
- [ ] I can test and roll back a control without exposing the whole organization.
- [ ] I understand the Free Tier and recurring-cost implications of a live landing zone.

## Official references

- [AWS Organizations documentation](https://docs.aws.amazon.com/organizations/)
- [AWS Control Tower documentation](https://docs.aws.amazon.com/controltower/)
- [AWS IAM Identity Center documentation](https://docs.aws.amazon.com/singlesignon/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html)
- [Organizing your AWS environment using multiple accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)

Continue to [cost, FinOps, and performance](20-cost-finops-and-performance.md).
