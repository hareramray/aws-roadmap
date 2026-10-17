# Plan migration waves from evidence and dependencies

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial turns a fictional portfolio into seven-R decisions, ordered waves, and a concrete server-rehosting runbook. The required work is local. A live AWS Transform MGN extension is appropriate only for a disposable supported server with explicit ownership and a cost estimate.

## 1. Build a portfolio that includes hidden dependencies

Create a worksheet with application, owner, business operation, runtime, data, dependencies, critical period, recovery target, current cost, licensing, and strategy. Use the following seven examples:

| Workload | Evidence | Initial strategy |
| --- | --- | --- |
| Unused campaign microsite | Owner confirms no traffic and records archived | Retire |
| Factory controller | Hardware timing and vendor support require local operation | Retain |
| Legacy intranet | Supported OS, little code change budget, deadline approaching | Rehost |
| VMware application group | Platform continuity is a documented constraint | Relocate, subject to current platform/licensing evaluation |
| Internal mail application | Business accepts a different managed product | Repurchase |
| Self-managed PostgreSQL | Application SQL compatible with selected managed engine | Replatform |
| Order fulfillment bottleneck | Measured scaling/correctness problem justifies redesign | Refactor/re-architect |

Every decision needs an owner and a revisit condition. Retain does not mean forget forever; retire requires data/contract/identity cleanup; repurchase still needs migration, integration, and user adoption. Do not label every virtual machine rehost before understanding its business use.

Interview operators about monthly jobs, file drops, vendor allowlists, hardware license servers, certificates, and manual steps. Validate inventory with observed configuration and traffic. A server that seems idle during one day may run a critical month-end process.

## 2. Model dependencies and detect impossible wave plans

Save `waves.py`:

```python
from graphlib import TopologicalSorter, CycleError

# Each key must wait for the named prerequisites to be ready.
dependencies = {
    "identity": set(),
    "network": set(),
    "logging": {"identity", "network"},
    "database": {"network", "identity", "logging"},
    "intranet": {"database", "identity", "logging"},
    "batch": {"database", "logging"},
    "business-validation": {"intranet", "batch"},
}
try:
    order = list(TopologicalSorter(dependencies).static_order())
except CycleError as error:
    raise SystemExit(f"DEPENDENCY CYCLE: {error}")
print(" -> ".join(order))
assert order.index("database") < order.index("intranet")
assert order.index("logging") < order.index("batch")
print("PASS: prerequisite ordering")
```

Run it and verify prerequisites precede consumers. Independent nodes may appear in different valid orders. Add `dependencies["identity"] = {"intranet"}` before sorting to demonstrate a cycle. Remove it after observing the failure.

Real business dependencies may be cyclic. A topological sort does not solve that by itself: co-migrate the group, introduce temporary connectivity, split a dependency, or redesign the interface. Record the chosen transition architecture and its expiration so temporary connections do not become permanent undocumented exposure.

## 3. Define assessment and mobilization outputs

Assessment produces the business case, portfolio data, baseline performance, dependency risks, and initial strategy. Mobilization produces a usable landing zone, access model, network/DNS, logging, security, backups, deployment artifacts, and trained operators.

For each wave, require entry criteria: identified owner, approved data handling, compatible target, tested backup, source health, capacity/quota checks, cost estimate, test plan, cutover window, rollback boundary, and communication plan. Exit criteria include business acceptance, measured performance, security/access tests, backup verification, monitoring, cost ownership, and handover.

Choose a representative low-impact first wave. Moving only trivial static servers can give false confidence about databases, directories, scheduled work, and licensing. Improve the runbook after the first wave before multiplying it across the estate.

## 4. Select current discovery and modernization paths

AWS Application Discovery Service and Migration Hub have new-customer availability restrictions. Do not make onboarding to those historical tools a required new lab. Use current AWS Transform capabilities, supported inventory imports, existing management tools, and partner discovery where suitable.

AWS Application Migration Service is now AWS Transform MGN. Its `mgn` namespace and older documents may retain familiar identifiers. Separate server replication from broader modernization assistance: code changes, containerization, schema conversion, and application behavior still need review and tests.

Treat machine-generated migration recommendations as proposals. Validate operating-system support, dependencies, utilization period, licensing, connectivity, and application-specific requirements. An apparently idle host might contain a recovery service that must remain available.

## 5. Write an MGN rehearsal runbook

For an optional disposable source, read current supported source prerequisites before installing an agent. Record source identity, OS, disks, data sensitivity, available outbound connectivity, and permissions. Decide whether the source is safe to replicate and whether its licenses permit the target use.

Prepare the target staging and launch configuration: account/Region, subnet, security groups, instance type, disks, encryption, IAM role, tags, and post-launch actions. Estimate replication/staging storage, servers, test targets, transfer, and logs. Agent-based replication and conversion do not mean every application dependency is copied automatically.

Begin replication using the documented procedure, monitor initial synchronization and ongoing lag, and resolve errors before a test launch. Launch a test instance into an isolated network with outbound behavior controlled: a cloned scheduled job must not send production emails, process real payments, or compete with its source.

Validate boot, service startup, volumes, application queries, identity, time synchronization, DNS, certificates, logging, and backups. Compare functional behavior and performance to the source baseline. Delete test targets using the documented cleanup path after recording evidence, while preserving the source and replication state needed for the planned migration.

The cutover procedure must specify write freeze, final synchronization, acceptance, target activation, monitoring, and source disposition. A “cutover ready” tool status is not business approval or proof of zero data loss.

## 6. Plan acceptance and handover

Create a matrix with source baseline, target expected value, actual observation, owner, and pass/fail. Include a representative user login, database write/read, scheduled report, outbound integration, denied unauthorized access, backup restore, and failure alert.

Document operations after migration: patch/image ownership, deployment process, incident escalation, recovery, capacity, cost review, and certificate/secret renewal. A rehosted application can be operationally worse if the old team assumes AWS now owns its guest OS and application maintenance.

For refactoring, split migration success from modernization success. Preserve a working baseline and introduce changes in testable increments. For repurchase, include data export/import, user provisioning, decommissioned integrations, and contractual obligations. For retirement, prove that required records and audit evidence remain accessible.

## Verification, troubleshooting, cost, and cleanup

Deliver all seven strategy decisions, the dependency model, one detected cycle, readiness gates, and an MGN test runbook. Explain which decisions are confirmed by evidence and which remain assumptions.

If migration lag never decreases, compare source change rate with effective replication throughput and inspect disk/network bottlenecks. If the clone boots but fails login, inspect directory, DNS, routes, time, and trust rather than repeating replication blindly. If the monthly batch fails, revisit discovery coverage and production calendar assumptions.

The baseline has no AWS charges. Optional live testing can create staging servers, volumes, snapshots, test instances, network services, and telemetry. Remove only dedicated test/replication resources according to the current tool workflow after deciding whether continued replication is needed. Remove agents and temporary permissions when the exercise ends. Never decommission a source merely because the copied instance starts.

References: [AWS seven-R strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [AWS Transform MGN](https://docs.aws.amazon.com/mgn/), [MGN rename](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-transform-mgn-rebrand/), [AWS maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
