# 21. Migration, hybrid, and edge

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete [networking](05-networking-and-dns.md), [compute](06-compute-and-load-balancing.md), [storage](07-storage-and-backup.md), [databases](08-databases-and-caching.md), [recovery](17-reliability-and-disaster-recovery.md), and [governance](19-organizations-and-governance.md).

Learn to assess a workload portfolio, choose migration strategies, move data with verification, plan a controlled cutover, and design hybrid or edge operation from real constraints. A successful migration preserves business behavior and establishes sustainable operations; copying virtual machines is only one possible step.

## 1. Discover the business system before moving it

Inventory applications, owners, servers, databases, file shares, scheduled jobs, identities, certificates, licenses, network flows, and external integrations. Include systems that are not visible in a server inventory: manually run scripts, vendor VPNs, printer connections, hardware license servers, and month-end processes.

Capture dependencies and timing. Two applications that communicate once a month can still require a coordinated migration wave. Interview operators and business owners, inspect configuration, review network observations, and validate assumptions against actual traffic. Treat discovery data as sensitive because it exposes the organization's internal architecture.

Assess business criticality, data classification, recovery objectives, operating-system support, application portability, contractual constraints, and target-region availability. Establish baseline performance and costs so that post-migration comparisons are meaningful.

AWS discovery and migration products have lifecycle changes. Application Discovery Service and Migration Hub are listed in maintenance, so a new learner should not build a required exercise around onboarding to them. Inspect current AWS Transform capabilities, supported exports, existing discovery tools, and partner options instead. [AWS services in maintenance](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).

## 2. Apply the seven migration strategies

| Strategy | Meaning | Example decision |
| --- | --- | --- |
| Retire | Remove an application that is no longer needed | Archive required records and decommission an unused reporting server |
| Retain | Keep it in its current environment for now | A hardware-dependent application remains on premises with a review date |
| Rehost | Move the workload with limited application change | Move a supported server to EC2 |
| Relocate | Move an existing platform environment with limited application redesign | Relocate suitable virtualized workloads while preserving platform operations |
| Repurchase | Replace the product or delivery model | Adopt an appropriate SaaS product with data and identity migration |
| Replatform | Make targeted platform changes without a full redesign | Move a self-managed supported database to a managed database |
| Refactor or re-architect | Redesign significant application behavior | Decompose a bottleneck and adopt event-driven processing |

These are the AWS seven Rs. They are portfolio decisions, not a ranking where refactoring is always best. A workload may use several strategies across components and phases. [AWS migration strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html).

Record the rationale, cost, risk, dependencies, and revisit conditions for each choice. Rehosting can meet a deadline but preserve expensive inefficiencies. Refactoring can create value but increase delivery risk. Retiring an unused system may produce more value than migrating it beautifully.

## 3. Organize assessment, mobilization, and migration waves

Use assessment to establish the business case and readiness. During mobilization, build the landing zone, network, access, logging, security, recovery, deployment process, and team skills. During migration and modernization, execute repeatable waves and improve the process after each one.

Group waves by dependencies, risk, and business timing. Start with representative but lower-impact workloads, not only trivial applications that teach nothing about the difficult cases. Define entry criteria and exit criteria: ownership, dependency mapping, backups, capacity, security approval where required, tests, cutover plan, rollback plan, and operating runbooks.

Build a migration factory only after a repeatable pattern is understood. Automate inventory validation, target provisioning, testing, evidence capture, and handover. Automation does not remove the need for a business owner to confirm that the migrated system behaves correctly.

## 4. Select the right transfer or migration mechanism

| Need | Starting point to evaluate | What still needs design |
| --- | --- | --- |
| Supported server rehosting | AWS Transform MGN | Replication network, launch settings, tests, licensing, cutover |
| Database move or change-data capture | AWS Database Migration Service and appropriate schema tooling | Schema compatibility, permissions, transaction consistency, validation |
| Bulk online file/object movement | AWS DataSync | Source permissions, bandwidth, metadata behavior, validation, scheduling |
| Ongoing file access across environments | Appropriate AWS Storage Gateway option | Cache sizing, failure behavior, protocols, and recovery |
| Managed partner file exchange | AWS Transfer Family | Authentication, partner onboarding, workflows, storage permissions |
| Application modernization | AWS Transform and supported modernization tools | Code review, regression tests, architecture, and operational readiness |

AWS Application Migration Service was renamed **AWS Transform MGN** in June 2026. Older tutorials and the `mgn` service namespace may retain the earlier name. Learn the actual replication, testing, and cutover workflow instead of assuming a renamed tool changes the application's requirements. [AWS rename announcement](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-transform-mgn-rebrand/).

For database migration, distinguish homogeneous from heterogeneous migration. Moving data does not automatically convert stored procedures, extensions, collations, indexes, privileges, and application SQL. CDC has source prerequisites and failure modes; monitor replication lag, errors, and validation throughout.

Estimate transfer feasibility:

```text
ideal transfer time = data size in bits / usable bits per second
actual time includes protocol overhead, source/destination limits,
competing traffic, retries, and changes generated during transfer
```

Measure available bandwidth rather than assuming a circuit's advertised speed is fully usable. Plan the initial copy and the change backlog separately. If the source changes faster than the replication pipeline can process, the backlog will not converge.

## 5. Design hybrid connectivity

Learn Site-to-Site VPN, Direct Connect, Transit Gateway, Cloud WAN, Direct Connect gateways, virtual interfaces, BGP, route propagation, and Route 53 Resolver endpoints. Use the simplest topology meeting the number of sites, isolation, bandwidth, latency, and failure requirements.

| Option | Typical role | Tradeoffs |
| --- | --- | --- |
| Site-to-Site VPN | Encrypted connectivity over appropriate underlying transport | Internet variability, tunnel configuration, throughput, failover |
| Direct Connect | Dedicated connectivity path through a provider/location | Provisioning lead time, redundancy, contracts, routing, encryption design |
| Transit Gateway | Regional hub for supported network attachments | Route-table segmentation, processing cost, shared failure impact |
| Cloud WAN | Policy-driven global network management | Global topology and policy complexity |

Direct Connect does **not** encrypt traffic by default. Evaluate supported MACsec, VPN over Direct Connect, and application TLS according to the required protection scope. Private connectivity and encrypted connectivity are different properties. [Direct Connect encryption](https://docs.aws.amazon.com/directconnect/latest/UserGuide/encryption-in-transit.html).

Plan nonoverlapping CIDRs, subnet growth, DNS forwarding, split-horizon naming, firewall rules, asymmetric routing, MTU, proxy behavior, and time synchronization. Hybrid outages often come from name resolution or routes rather than the application code. Test primary-path failure and confirm backup-path capacity and routing convergence.

Two links are not independent if they share the same building, provider path, router, power, or configuration error. Model physical and logical failure domains. Document who operates the on-premises side, who contacts the carrier, and how an incident is diagnosed across teams.

## 6. Understand VMware, on-premises AWS, and edge choices

For VMware estates, compare rehosting to native EC2, replatforming applications, retaining existing infrastructure, and Amazon Elastic VMware Service (EVS) where suitable. EVS supports VMware Cloud Foundation within a customer VPC; evaluate licensing, supported versions, hardware/capacity requirements, networking, management responsibility, and partner operations. It is not an inexpensive beginner lab. [Amazon EVS](https://aws.amazon.com/evs/).

Do not assume historical VMware Cloud on AWS purchasing arrangements still apply. AWS stopped reselling that offering in 2024; review current Broadcom and AWS guidance for an existing environment before choosing a contractual or technical path. [AWS VMware transition guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-from-vmware-to-the-aws-cloud/).

Compare edge options by what must remain local:

- **AWS Outposts:** AWS infrastructure at a customer site for supported workloads with local latency or residency needs. Plan facility, networking, service-link, capacity, physical security, and disconnected-operation limitations.
- **Local Zones:** selected AWS infrastructure closer to some population or industry centers. Verify service coverage and parent-Region dependencies.
- **AWS Wavelength:** supported telecommunications edge scenarios where carrier placement is relevant; verify locations, availability, and architecture requirements.
- **CloudFront and edge functions:** content delivery and suitable request transformations close to users, not a substitute for a full local data center.
- **IoT Greengrass V2 or other local runtime:** device/industrial software that needs local processing and intermittent connectivity; design updates, identity, buffering, and eventual synchronization.

Choose local processing only when latency, bandwidth, data location, or disconnected-operation needs justify the management burden. Hardware at the edge still needs patching, replacements, access control, monitoring, and an end-of-life plan.

Do not select Snowball from an old roadmap without checking its lifecycle. The current product notice states commercial-Region support ends December 31, 2026, with specified exceptions. This roadmap treats Snow hardware as a lifecycle/migration topic rather than a new beginner project. [AWS Snowball notice](https://aws.amazon.com/snowball/).

## 7. Execute cutover and plan rollback

A cutover runbook should include timestamps, owners, communication, write-freeze rules, final synchronization, validation, traffic changes, monitoring, and explicit go/no-go criteria. Reduce DNS TTLs in advance where useful, but account for caches and clients that reuse connections.

Rollback is a data problem as well as a routing problem. If the target accepts writes, returning users to the source can lose those writes unless reverse synchronization or reconciliation has been designed. State the latest point when a simple rollback remains safe and what happens afterward.

After cutover, keep focused monitoring through a defined stabilization period. Compare latency, errors, batch completion, security findings, backups, cost, and user acceptance to the baseline. Decommission the source only after the agreed acceptance and retention conditions are met.

## Practical exercise: migrate a small application dataset

**Scope:** a local disposable application with synthetic data, plus a design-only target or a small sandbox target. No production data or enterprise network changes are needed.

1. Inventory the application's runtime, data, identities, dependencies, scheduled work, and owner.
2. Assign a seven-R strategy and explain one rejected alternative.
3. Build a dependency diagram and a target design, including networking, logs, backup, and cost assumptions.
4. Export or replicate a small dataset using an appropriate mechanism. Record counts, checksums, constraints, and a latest transaction identifier.
5. Start the target with a test endpoint and run functional, permission, and failure tests.
6. Rehearse a write freeze, final synchronization, validation, and traffic switch. In a local simulation, represent traffic switching with configuration rather than public DNS.
7. Rehearse rollback before accepting target writes, then separately describe the harder case after new writes.
8. Write a handover document with ownership, alerts, recovery, cost, and decommission criteria.

**Verification:** prove record equivalence, application behavior, denied unauthorized access, and a working backup/restore path. Record measured copy duration and explain why production timing may differ.

**Cost and cleanup:** estimate temporary double-running, replication resources, databases, gateways, storage, logs, and transfer. Delete target lab resources, replication tasks, agents, temporary credentials, staging storage, and retained snapshots when no longer needed. Do not delete the source until verification is complete. Direct Connect, Outposts, and enterprise VMware should remain design exercises unless an actual project authorizes their cost and commitments.

## Mastery checklist and references

- [ ] I can apply all seven Rs with business reasoning.
- [ ] I can distinguish server, database, file-transfer, and ongoing hybrid-access tools.
- [ ] I can explain a complete cutover and the data consequences of rollback.
- [ ] I can troubleshoot routes, DNS, encryption, and hybrid failover.
- [ ] I can justify edge placement and its operational dependencies.
- [ ] I can identify obsolete migration advice before using it.

- [AWS migration guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/welcome.html)
- [AWS Transform MGN documentation](https://docs.aws.amazon.com/mgn/)
- [AWS DMS documentation](https://docs.aws.amazon.com/dms/)
- [AWS DataSync documentation](https://docs.aws.amazon.com/datasync/)
- [Hybrid connectivity whitepaper](https://docs.aws.amazon.com/whitepapers/latest/hybrid-connectivity/hybrid-connectivity.html)
- [AWS Outposts documentation](https://docs.aws.amazon.com/outposts/)

Continue to [specialized domains](22-specialized-domains.md).
