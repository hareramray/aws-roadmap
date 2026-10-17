# Migration, hybrid, and edge tutorials

[All tutorials](../README.md) · [Source roadmap](../../21-migration-hybrid-and-edge.md)

| Tutorial | Deliverable |
| --- | --- |
| [01 — Discover a portfolio and plan migration waves](01-portfolio-and-migration-waves.md) | Seven-R decisions, dependency order, readiness gates, and MGN test/cutover plan |
| [02 — Reconcile data and rehearse cutover](02-data-migration-and-cutover.md) | Runnable snapshot/CDC model, validation, rollback boundary, and transfer estimate |
| [03 — Design hybrid connectivity and edge operation](03-hybrid-network-and-edge-design.md) | CIDR checks, routes/DNS/failure tests, and VMware/Outposts/edge decision record |

## Coverage mapping

| Original heading or service family | Worked coverage |
| --- | --- |
| Discovery | 01 inventory, owners, dependencies, baseline, lifecycle-aware tooling |
| Seven Rs | 01 explicit decision for each strategy and revisit condition |
| Assessment/mobilization/waves | 01 readiness gates, topological ordering, acceptance and handover |
| Server/application migration | 01 AWS Transform MGN, current AWS Transform, testing, launch settings |
| Database and file movement | 02 DMS/schema conversion, CDC, DataSync, Transfer Family, Storage Gateway |
| Transfer feasibility | 02 bandwidth calculation, change backlog, reconciliation |
| Hybrid network | 03 VPN, Direct Connect, Transit Gateway, Cloud WAN, gateways/VIFs/BGP, DNS |
| VMware and edge | 03 EVS/VCF, VMware Cloud on AWS commercial context, Outposts, Local Zones, Wavelength |
| Edge software/hybrid containers | 03 CloudFront, Greengrass V2, EKS Hybrid Nodes/Anywhere, ECS Anywhere |
| Snow lifecycle | 03 current availability check and alternative transfer planning |
| Cutover/rollback/decommission | 02 write freeze, watermark, target writes, rollback and handover |
| Exercises/pitfalls/mastery | Runnable local checks and concrete failure/cleanup sections in all three |

Start locally. Dedicated circuits, enterprise VMware, Outposts hardware, and production cutovers are design exercises here. No AWS resources, network changes, or commercial commitments were made while creating this documentation.
