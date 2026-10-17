# Design a landing zone and prove what an SCP does

[Domain index](README.md) · [All tutorials](../README.md)

The required exercise is design-only. It creates an account structure, an access matrix, a narrow policy, and a concrete verification plan. The optional live section applies only to an organization already intended for experimentation. Never attach a new broad policy to an organization root as a learning shortcut.

## 1. Create an account and OU inventory

Use a fictional company with an application team, a platform team, and a security team. Write `accounts.csv`:

```csv
account_alias,ou,owner,environment,purpose
management,root,platform,administration,organization and billing
log-archive,security,security,shared,protected audit evidence
security-tools,security,security,shared,delegated security administration
network,platform,platform,shared,shared network resources
orders-dev,workloads-nonprod,orders,development,order application testing
orders-prod,workloads-prod,orders,production,customer order application
policy-test,policy-staging,platform,test,organization control verification
```

Do not use fabricated twelve-digit IDs as if they were live accounts. Aliases are sufficient for the design. Each account needs an owner, purpose, cost assignment, contact, lifecycle state, and intended control baseline.

Explain why production and development are separate: access, change cadence, quotas, incident scope, and billing ownership differ. Explain why the management account hosts no application: its organization responsibilities and guardrail exceptions warrant tightly constrained use. An OU groups similar control requirements; it need not mirror the human organization chart.

Add a quarantine OU only after defining evidence preservation and responder access. Moving a compromised account under a broad deny can also interrupt investigation, logs, or recovery. Test the exact required actions before treating quarantine as a universal emergency button.

## 2. Write the human-access matrix

Create permission sets or equivalent role designs for Developer, ProductionReadOnly, Deployment, SecurityInvestigator, NetworkOperator, BillingAnalyst, and EmergencyAdministrator. Assign groups rather than individual names where possible. Specify account scope, session duration, authentication, allowed actions, and review owner.

The developer can inspect development and request a deployment; the pipeline deployment role performs production changes within its scope. Security investigators can read relevant evidence but should not casually alter retention. Billing analysts need cost data, not application secrets.

Use Identity Center organization-instance access where appropriate. An account instance for application assignments is not a substitute for organization-wide AWS account access. Plan identity-provider lifecycle removal and group synchronization. Test a user removal and a revoked session according to the applicable identity system's behavior.

Document emergency access: who holds it, how it is authenticated, which outage it survives, who is notified, how actions are audited, and how the session/permissions are removed after use. Review current centralized root-management capabilities separately; do not assume an SCP protects the management account.

## 3. Draft one narrow SCP

The exercise control prevents member-account principals from stopping a trail. It is intentionally narrower than a complete protected-logging policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PreventStoppingTrailInTestAccount",
      "Effect": "Deny",
      "Action": "cloudtrail:StopLogging",
      "Resource": "*"
    }
  ]
}
```

This SCP grants no action. Applicable IAM permissions are still needed for an allowed operation. Keep the organization's required allow policies in place; removing every allow in an allow-list design can deny much more than intended. A member-account administrator cannot override an applicable explicit SCP deny by attaching AdministratorAccess.

This policy is not full logging protection. It does not by itself prevent trail deletion, destination-policy changes, key denial, retention changes, or malicious application logging omissions. Its purpose is to let you test one rule precisely before expanding scope.

## 4. Predict request outcomes

Build the following test table. Evaluate it manually against the documented policy rules rather than writing a toy script and calling it a complete IAM simulator.

| Principal and request | Expected outcome | Reason |
| --- | --- | --- |
| Test-account role with IAM allow, `StopLogging` | Denied | Applicable SCP explicit deny |
| Same role with IAM allow, `GetTrailStatus` | Allowed if all other controls permit | SCP does not deny this action |
| Role without IAM allow, `GetTrailStatus` | Denied | SCP absence of deny does not grant IAM permission |
| Management-account role | Evaluate its IAM/resource controls separately | Member-account SCP does not constrain it |
| External principal accessing a resource | Evaluate actual principal/resource context | A resource owner's SCP is not a universal external-principal boundary |

For the final case, learn resource policies and RCPs. RCPs restrict supported member-account resources and can constrain relevant external access, but do not grant access or cover every service. Check documented exceptions and supported actions. Do not paste a blanket organizational deny into a production organization before testing service-to-service paths.

Also distinguish permissions boundaries, session policies, and trust policies. A boundary constrains an identity's possible permissions; a trust policy governs role assumption; a session policy constrains a session. Resource-policy grants to different principal forms have nuances, so avoid reducing every request to a universal union/intersection slogan.

## 5. Relate the rule to Control Tower

A preventive control blocks defined actions/behavior. A detective control identifies noncompliant configuration. A proactive control checks supported provisioning requests before creation; Control Tower's proactive controls use CloudFormation hooks rather than intercepting every possible service API.

For the logging objective, document one preventive rule, one detective signal for missing/unhealthy delivery, and one response runbook. A trail can be configured but fail to deliver because of destination permissions. Compliance configuration and operational delivery are distinct checks.

Control Tower provides landing-zone and account-governance workflows. Decide the desired home/governed Regions, accounts, integrations, and ownership before enrollment. Verify exact control support and prerequisites. Do not create duplicate custom resources that conflict with a managed landing-zone baseline.

## 6. Optional live verification

First inspect current account-plan/Free Tier implications. The current FAQ says earning activities expire and new credit earning ends on joining an organization or setting up Control Tower; cohort-specific treatment of existing credits and plan features needs separate verification.

Use an empty policy-staging member account. Create a dedicated tutorial trail only if its destinations and cost are understood. Attach the SCP to that staging scope, then perform the intended denied operation with the actual test role. Record principal ARN, request ID, time, expected result, and observed denial. Test the allowed read using the same identity.

Confirm rollback from the authorized organization administration path before attachment: detach the policy from the staging OU/account, then retest. Do not depend on the denied member-account role to remove its own organizational restriction. Policy syntax validation is useful but does not replace actual request testing.

## Verification, troubleshooting, cost, and cleanup

Deliver the OU/account inventory, permission matrix, SCP, request table, control register, and rollback instructions. A reviewer should explain why the SCP cannot grant missing read permission and why the management account needs its own protection.

For unexpected denial, inspect identity, trust/session context, resource policy, key policy, boundary, endpoint policy, SCP/RCP, and service prerequisites. Do not “fix” it by adding administrator access blindly. For unexpected allow, confirm attachment scope and whether the request is from an affected principal.

Design-only work incurs no AWS resource charges. A live extension can add CloudTrail storage/events, Config, security integrations, logs, and other baseline costs. Detach/delete only the temporary policy, delete only the dedicated tutorial trail and its disposable destination data, and remove temporary roles. Closing accounts or dismantling a landing zone is a separate lifecycle operation, not routine lab cleanup.

References: [SCP evaluation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html), [RCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html), [Control Tower control behavior](https://docs.aws.amazon.com/controltower/latest/controlreference/control-behavior.html), [Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/).
