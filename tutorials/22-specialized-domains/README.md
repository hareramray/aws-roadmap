# Specialized-domain tutorials

[All tutorials](../README.md) · [Source roadmap](../../22-specialized-domains.md)

These are worked domain exercises with local simulations and bounded optional service extensions. They do not require industrial hardware, phone calls, desktop fleets, quantum hardware, satellite contacts, or enterprise purchases. Simulations teach contracts and validation; they are explicitly not evidence of a deployed AWS service.

| Tutorial | Worked outcome |
| --- | --- |
| [01 — IoT, industrial state, and geofences](01-iot-industrial-and-geospatial.md) | Per-device authorization, duplicate/reconnect handling, asset model and location transitions |
| [02 — Media, games, rendering, and desktops](02-media-games-and-workspaces.md) | Media pipeline/test asset, streaming economics, game capacity and end-user delivery design |
| [03 — Contact, business, and industry integrations](03-contact-business-and-industry.md) | Contact flow, delivery/reconciliation contract, EDI/SaaS mapping and industry control matrix |
| [04 — HPC, quantum, space, and legacy migration](04-hpc-quantum-space-and-legacy.md) | Parallel-work benchmark, quantum simulation, satellite pipeline and lifecycle migration assessment |

## Coverage mapping

| Original heading and named branches | Tutorial |
| --- | --- |
| Lifecycle-aware selection | All; 04 explicit migration register and acceptance gates |
| IoT and industrial systems | 01 MQTT, IoT Core, shadows, Device Management, certificates, Greengrass V2, SiteWise, TwinMaker, feature lifecycle |
| Media | 02 MediaConvert, MediaLive, MediaPackage, MediaConnect, CloudFront, IVS, MediaTailor, Deadline Cloud |
| End-user computing | 02 WorkSpaces, WorkSpaces Applications, image/profile/persistence/licensing, current variant restrictions |
| Contact center and communications | 03 Connect, SES, End User Messaging, supported Chime SDK, Pinpoint lifecycle |
| Business integration | 03 AppFlow, EventBridge/APIs, Transfer Family, B2B Data Interchange, Quick/ERP and reconciliation |
| Industry workloads | 03 healthcare/HealthLake/HealthImaging/HealthOmics, supply chain/Connect Decisions, financial services, SAP/mainframe, automotive/manufacturing/telecom |
| HPC/scientific computing | 04 Batch, PCS, ParallelCluster, MPI/Slurm, EFA, FSx Lustre, accelerators, checkpointing |
| Quantum | 04 Braket, local circuit, noise/shots, hybrid workflow and remote-cost controls |
| Blockchain/ledger | 04 trust/finality/keys, QLDB/Managed Blockchain lifecycle and alternatives |
| Games | 02 GameLift Servers, GameLift Streams, matchmaking/session capacity/state/abuse |
| Geospatial | 01 Location Service maps/places/routes/tracking/geofencing and privacy |
| Space | 04 Ground Station contacts/antennas/downlink and processing validation |
| Capstone/pitfalls/mastery | Each tutorial ends with evidence, failure tests, costs and cleanup |

Use current primary documentation before every optional deployment. The source review date is 2026-10-18; service and feature availability can change independently. A retired feature is covered through migration assessment, never through a new deployment requirement.
