# Reconcile a small bill and assign its shared costs

[Domain index](README.md) · [All tutorials](../README.md)

You will analyze a synthetic bill locally, allocate shared services, and write a budget/anomaly response. This provides a complete calculation before touching real billing data. Python is sufficient; an AWS billing export is an optional read-only extension.

## 1. Build the synthetic input

Save `costs.csv`:

```csv
account,application,category,amount
orders-prod,orders,compute,120.00
orders-prod,orders,database,80.00
catalog-prod,catalog,compute,80.00
catalog-prod,catalog,storage,20.00
platform,shared,network,60.00
platform,shared,observability,40.00
```

The amounts are fictional accounting units in one common currency. They are not service rate quotes. Assume 75,000 successful orders-platform operations and 25,000 catalog-platform operations during the period. This chosen allocation basis is useful for the exercise, but a real networking allocation might be more defensible using measured traffic.

Before calculating, state the cost view: these are synthetic net operating charges, with no refunds, tax, support, upfront commitment payments, or credits. Real exports require explicit handling of those line types. Comparing an amortized report with an unblended invoice without explaining the difference can produce a false discrepancy.

## 2. Allocate using exact decimal arithmetic

Save `allocate.py`:

```python
import csv
from collections import defaultdict
from decimal import Decimal
from pathlib import Path

direct = defaultdict(lambda: Decimal("0"))
shared = Decimal("0")
source_total = Decimal("0")
with Path("costs.csv").open(newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        amount = Decimal(row["amount"])
        source_total += amount
        if row["application"] == "shared":
            shared += amount
        else:
            direct[row["application"]] += amount

units = {"orders": 75000, "catalog": 25000}
all_units = sum(units.values())
allocated = {}
for app, count in units.items():
    share = shared * Decimal(count) / Decimal(all_units)
    allocated[app] = direct[app] + share
    print(app, "direct", direct[app], "shared", share,
          "total", allocated[app], "per_success", allocated[app] / Decimal(count))
assert sum(allocated.values()) == source_total
print("RECONCILED", source_total)
```

Expect orders direct 200, shared 75, total 275; catalog direct 100, shared 25, total 125; grand total 400. Orders cost per successful operation is approximately 0.0036667 synthetic units; catalog is 0.005. These metrics are not directly comparable if the operations deliver different value or consume different work.

Change one input application to `unassigned` and rerun. The reconciliation assertion should fail because the script has no allocation destination for that direct charge. This is useful: unmapped costs should be visible, not silently discarded. Add an explicit unallocated bucket in a production report and assign an owner to fix it.

## 3. Define an allocation policy people can explain

Record the treatment of shared networking, security, observability, support, and commitments. Choose showback if teams need information and chargeback if financial responsibility is transferred. Document why the allocation basis is fair enough for the decision being made.

Add tags such as application, owner, environment, and cost center where supported. Use account ownership and Cost Categories to cover dimensions that tags alone cannot represent. Not every line item is taggable, and activating a tag does not guarantee historical records acquire it retroactively. Inspect actual export behavior.

Avoid incentives that reward harmful behavior. Allocating all security costs to the team reporting the most findings could discourage investigation. Allocating shared infrastructure equally may be simple but unfair for radically different usage. Review the policy with engineering and finance rather than presenting it as an objective law.

## 4. Translate to AWS billing tools

Use Cost Explorer to inspect daily service/account/usage-type trends. Choose the same cost view and date boundary used by your report. For deeper analysis, evaluate AWS Data Exports and CUR 2.0. Select the needed columns and storage destination, then inspect the delivered schema rather than copying a query written for a different export generation.

Normalize a small export sample into the local columns used above, preserving original line identifiers and cost type for auditability. Reconcile the normalized total to the selected source report before allocating. Account for adjustments, refunds, credits, taxes, and commitment treatment deliberately.

Protect exports as business-sensitive data. Limit bucket and query access, avoid public dashboards, and retain only what the reporting purpose needs. Storage, Athena queries, and BI subscriptions used to analyze billing may themselves be billable. A “read-only analysis” can still incur query charges.

## 5. Estimate the next month with scenarios

Separate baseline and variable drivers. For each workload, record operating hours, requests, average execution duration/memory, database capacity, storage growth, backup retention, transfer, and telemetry. Use current regional prices in AWS Pricing Calculator.

Build three forecasts: expected traffic, a campaign peak with a defined duration, and a misconfiguration case such as repeated retries or an unbounded scan. Include temporary double-running during migration and deployment. Do not multiply a daily peak by every hour of the month unless that is the intended scenario.

Compare total cost of ownership, including staffing, patching, incidents, licenses, and recovery work. A low infrastructure rate does not necessarily mean a lower operating cost. Record uncertainty explicitly, especially when data volume or query patterns are not yet measured.

## 6. Build a safe anomaly response

Assume daily network charges rise from 2 to 18 synthetic units immediately after a deployment. Write a hypothesis table: new cross-AZ path, NAT traffic, internet transfer, duplicate data copy, or legitimate traffic increase. Inspect account, Region, usage type, resource inventory, deployment timeline, and request/byte metrics.

Set actual and forecast budget alerts at meaningful account/workload scopes and test the destination you control. AWS Budgets has delayed billing/notification behavior and is not a universal hard spending cap. Budget actions restrict selected behavior; they do not erase charges, stop every resource, or cancel commitments.

Keep this separate from account/project spend-limit features in newer signup experiences. Check the exact account's supported plan controls. Neither “all accounts are capped” nor “no account can have a spending control” is a safe blanket assumption.

The response runbook should first identify the owner and workload impact. For a disposable sandbox, stop the faulty generator or restrict further provisioning. For production, choose a mitigation that preserves required service and data. Automatically shutting down a database solely because an alert fired can create a larger business loss.

## Verification, troubleshooting, cost, and cleanup

Acceptance requires exact reconciliation to 400, documented shared allocation, an intentionally detected unmapped charge, and an anomaly runbook with a safe mitigation. Explain why per-success cost increases when failures consume resources without creating useful output.

If real totals differ, check time zone/date boundaries, billing refresh, cost view, excluded line types, refunds, tag coverage, and duplicated export partitions. If a budget never alerts, verify filter scope, actual versus forecast settings, notification confirmation, and data arrival. Do not generate large real spend merely to test a notification.

The local exercise has no cloud costs. If you created an export, dashboard, or query workspace just for the tutorial, stop recurring delivery where appropriate and remove dedicated storage/query outputs and subscriptions after preserving redacted evidence. Keep useful account budget alerts active. Recheck delayed billing before declaring a cloud lab fully cleaned up.

References: [AWS Data Exports](https://docs.aws.amazon.com/cur/latest/userguide/what-is-data-exports.html), [Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html), [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html), [Free Tier account terms](https://aws.amazon.com/free/free-tier-faqs/).
