# Build an account request and govern what the account shares

[Domain index](README.md) · [All tutorials](../README.md)

This exercise implements a local account-request validator and turns its output into a provisioning checklist. You then design shared resources, evidence collection, inventory, exceptions, and retirement. It teaches the work around account creation without creating billable organization infrastructure.

## 1. Make the account request complete

Save `account-request.json`:

```json
{
  "alias": "orders-test",
  "owner": "orders-team",
  "environment": "test",
  "cost_center": "engineering",
  "data_classification": "synthetic",
  "regions": ["ap-south-1"],
  "network_mode": "isolated",
  "expires_on": "2027-01-31",
  "contact": "owner@example.invalid"
}
```

The reserved `.invalid` address is fictional and must never be submitted as an actual account email. The example date is a design input, not a scheduled action. For a real request, verify uniqueness, organization policy, account quota, contact ownership, and the current account-plan implications.

Save `validate_request.py`:

```python
import json
import re
from datetime import date
from pathlib import Path

r = json.loads(Path("account-request.json").read_text())
required = {"alias", "owner", "environment", "cost_center", "data_classification",
            "regions", "network_mode", "expires_on", "contact"}
errors = []
if set(r) != required:
    errors.append("request must contain exactly the documented fields")
if not re.fullmatch(r"[a-z][a-z0-9-]{2,29}", str(r.get("alias", ""))):
    errors.append("invalid alias")
if r.get("environment") not in {"development", "test", "production"}:
    errors.append("unknown environment")
if r.get("data_classification") not in {"synthetic", "internal", "restricted"}:
    errors.append("unknown classification")
if r.get("network_mode") not in {"isolated", "shared-vpc", "dedicated-vpc"}:
    errors.append("unknown network mode")
if not isinstance(r.get("regions"), list) or not r["regions"]:
    errors.append("at least one intended Region is required")
try:
    date.fromisoformat(r.get("expires_on", ""))
except ValueError:
    errors.append("expiry must be an ISO date")
for key in ("owner", "cost_center", "contact"):
    if not r.get(key):
        errors.append(f"missing {key}")
if errors:
    raise SystemExit("REJECTED: " + "; ".join(errors))
print("ACCEPTED FOR DESIGN REVIEW; not submitted to AWS")
print(json.dumps({"account": r["alias"], "baseline": "tutorial-v1",
                  "owner": r["owner"], "network": r["network_mode"]}, indent=2))
```

Run it, then remove the owner and rerun. The first request should pass local review; the second should fail. Restore the valid input. The script validates only this tutorial's schema, not email deliverability, organizational permission, legal suitability, or AWS account creation.

## 2. Turn approval into a versioned baseline

Define baseline `tutorial-v1` with account metadata, workforce role assignments, deployment access, logging destinations, security enrollment, allowed Regions, cost tags, budgets, recovery settings, and ownership contacts. Record which team operates each baseline component.

Use Control Tower Account Factory or an appropriate account-vending automation when implementing in an established organization. Terraform-based account customization is a separate supported workflow to evaluate, not a reason to allow several tools to manage the same resource independently.

Create an acceptance checklist: account reachable by the intended permission set; prohibited role denied; audit event arrives in the log archive; selected security coverage is enabled in intended Regions; billing ownership is visible; baseline version is recorded; no unintended public resource exists. Account creation status alone does not prove readiness.

Test existing-account enrollment separately from new-account provisioning. Existing networking, recorders, roles, and policies can conflict with the baseline. Inventory before adoption, plan changes, and retain a rollback/recovery path.

## 3. Design central evidence and security ownership

Draw workload account → supported organization trail/configuration/security collection → security/log accounts. Separate evidence writers, readers, retention administrators, and responders. Specify destination policy, encryption permissions, retention, alerting on delivery failure, and recovery access if the workload account is compromised.

For supported GuardDuty, Security Hub, Inspector, and Macie integrations, record delegated administrator, Regions, account enrollment, selected protection features, and cost owner. Central aggregation does not automatically enable every underlying feature. Test one representative finding/configuration change and follow it to the intended investigator.

Define backup policy ownership and test actual backup/restore results. A central backup policy does not prove every resource supports every copy or restore mode. Tag policies standardize supported tagging behavior; they are not universal guarantees that all resources have cost tags. Declarative policies configure supported service behavior and require their own scope/inheritance review.

## 4. Share resources deliberately with AWS RAM

Suppose the platform account owns a VPC and shares selected subnets with workload accounts. Use the supported RAM resource-sharing model and document what remains owned by the network account versus participating workload accounts. Sharing a subnet does not transfer the entire network's administration or grant arbitrary application IAM permissions.

Write a sharing matrix with resource, owner, consumer, permission, purpose, expiration/review, and revocation effect. Add tests: intended account can use the supported shared resource; unrelated account cannot; removing a share has a documented impact on existing and new usage. Check resource-specific behavior before assuming revocation immediately deletes consumer resources.

RAM shares supported resource types, not arbitrary objects. For resources outside that support, evaluate service-specific resource policies or another architecture. Cross-account sharing must be visible in incident and retirement inventories.

## 5. Provide approved provisioning with Service Catalog

Create a design for an approved development environment product: a private bucket, a small bounded worker, retention, owner tags, and a budget reference. Define allowed parameters and versioned templates. Use Service Catalog products/portfolios and launch constraints where suitable so consumers can provision approved infrastructure through controlled roles.

A catalog is not a substitute for reviewing the template or constraining the launch role. Test a valid parameter set, an invalid size/environment, an upgrade, and termination with retained data. Document who patches the product, who communicates deprecation, and which consumer versions remain deployed.

For a local demonstration, represent two product versions as JSON and write a change table: added logging retention, narrowed role scope, and a compatible parameter default. Review the resulting resource changes before upgrading consumers. Do not silently replace a stateful resource because its template version changed.

## 6. Inventory drift and manage exceptions

Use Resource Explorer where appropriate for supported resource discovery, together with tags, service inventories, Config, and billing exports. No single inventory is automatically complete for every service or Region. Reconcile the account registry with actual deployment ownership and charges.

Create `exceptions.csv` with exception ID, control, account/resource, business reason, compensating control, owner, approver, and expiry. A time-limited public endpoint exception, for example, needs authentication, monitoring, and a closure test. An exception with no owner or expiry should fail review.

Define drift handling: detect the mismatch, identify the authoritative manager, assess business impact, decide whether code or live state should change, review the plan, and verify convergence. Automatically overwriting every manual incident fix can worsen recovery; reconcile emergency changes deliberately.

## 7. Rehearse retirement without closing an account

For `orders-test`, stop new provisioning, inventory resources/shares, export required evidence, preserve obligated data, revoke unnecessary access, terminate owned workloads, and verify costs after billing updates. Update contacts and ownership throughout the process.

Before actual account closure or leaving an organization, check AWS requirements, retention, payment responsibility, remaining subscriptions, shared dependencies, and recovery windows. Closing an account is not the first cleanup step, and moving it to a suspended OU does not necessarily stop all charges.

## Verification, cost, and cleanup

Deliver a validated request, baseline acceptance checklist, control register, sharing matrix, catalog product lifecycle, and retirement plan. Demonstrate the validator's missing-owner failure and describe a negative cross-account access test. Explain who can recover when the identity provider or logging destination is unavailable.

Local work incurs no AWS resource cost. Optional live work can incur Config, trail storage/events, security services, catalog-created resources, shared networking, and inventory-related costs. Remove only dedicated tutorial shares, products, versions, and workloads after checking consumers. Preserve shared baselines and evidence according to policy.

References: [Control Tower Account Factory](https://docs.aws.amazon.com/controltower/latest/userguide/account-factory.html), [AWS RAM](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html), [Service Catalog](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/introduction.html), [Resource Explorer](https://docs.aws.amazon.com/resource-explorer/latest/userguide/welcome.html), [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html).
