# 22. Specialized AWS domains

[Roadmap index](README.md) · Reviewed: 2026-10-18

## Prerequisites and outcomes

Complete the core roadmap before specializing. In particular, understand identity, networking, event-driven integration, data protection, observability, cost, and recovery. Add domain knowledge: MQTT and embedded systems for IoT, codecs for media, numerical methods for HPC, or industry data standards for healthcare.

This module maps specialist areas to the concepts and evidence required to work in them. Its purpose is to help you recognize a suitable domain, investigate the current service offering, and complete a small demonstrable project. Specialized expertise comes from understanding domain constraints, not memorizing a long product catalog.

## 1. Use a lifecycle-aware service-selection process

Before starting a tutorial, verify the product's current availability, supported Regions, new-customer access, feature lifecycle, quotas, pricing dimensions, and migration guidance. A page remaining online does not prove that new customers can use the service. A feature can retire while the parent service remains available.

AWS publishes separate lists for [maintenance](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html), [sunset](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html), and [full shutdown](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html). At this review date, examples that should not become new required roadmap projects include QLDB, IoT Analytics, IoT Events, WorkDocs, Elastic Transcoder, and MediaStore, which appear in the shutdown list. Check the service-specific notice when an old course uses them.

For every domain, answer five questions: What is the business outcome? What must happen locally or in real time? What data is sensitive? What can fail? Which usage dimension can create unexpected cost?

## 2. Internet of Things and industrial systems

Start with devices, identities, telemetry, commands, and intermittent connectivity. Study MQTT topics, subscriptions, quality-of-service behavior, retained messages, persistent sessions, client identifiers, and reconnect behavior. Understand supported protocol behavior rather than assuming an MQTT broker has every possible extension.

AWS IoT Core provides managed connectivity and messaging, with rules that route device data to other services. Device shadows represent desired and reported state; they are not a full historical telemetry store. IoT policies govern device actions and topics, while IAM governs relevant human and service administration. [AWS IoT Core](https://aws.amazon.com/iot-core/).

Learn certificate provisioning, per-device identity, secure key storage, rotation, revocation, fleet inventory, firmware versions, deployment cohorts, and over-the-air update rollback. Never distribute one unrestricted credential to an entire fleet. A compromised device should not be able to impersonate every other device or publish to unrelated tenants' topics.

Use Greengrass V2 when local components need to execute and coordinate with AWS under intermittent connectivity. Design buffering, deduplication, time synchronization, storage exhaustion, and reconnect bursts. An industrial control loop with safety-critical timing should not depend on a remote cloud round trip.

For industrial information, evaluate IoT SiteWise for asset models and operational data, and IoT TwinMaker where a digital-twin view helps integrate systems. Distinguish a dashboard representation from the physical process itself. Verify feature-level lifecycle: SiteWise Monitor and its Data Processing Pack, IoT FleetWise, and Device Defender Detect have maintenance notices; learn the currently supported alternatives rather than assuming every historical IoT feature is available. [AWS maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).

**Domain exercise:** simulate two temperature sensors with separate credentials and topic permissions. Route synthetic readings to a small storage target, detect a missing-heartbeat condition, and reconnect after a local interruption. Verify one device cannot publish as the other. Revoke credentials and delete rules, things, storage, and logs after the exercise; message volume, rules, device management, and retained data can be billable.

## 3. Media, streaming, and content production

Learn the pipeline before the service names: ingest, encode/transcode, package, protect, originate, distribute, play, and measure. Understand codecs, bitrate, frame rate, resolution, adaptive bitrate ladders, HLS/DASH, manifests, segments, captions, audio tracks, and end-to-end latency.

Compare the main building blocks:

| Need | AWS direction to investigate | Key decision |
| --- | --- | --- |
| File-based video processing | AWS Elemental MediaConvert | Output ladder, codec compatibility, quality, and processing cost |
| Live broadcast encoding | AWS Elemental MediaLive | Input redundancy, channels, output groups, and operations |
| Packaging and origin behavior | AWS Elemental MediaPackage | Supported delivery formats, protection, latency, and availability |
| Professional live transport | AWS Elemental MediaConnect | Transport reliability, routing, source authorization, and failover |
| Interactive live streaming | Amazon IVS | Latency, audience scale, interaction, and moderation |
| Global content delivery | CloudFront with an appropriate origin | Cache behavior, origin access, geographic delivery, and transfer |
| Rendering workloads | AWS Deadline Cloud | Software licensing, worker sizing, assets, and job scheduling |

These are distinct roles rather than interchangeable streaming products. Compare an integrated service with a composable broadcast pipeline according to operational skill, required control, and audience experience. [AWS Media Services](https://aws.amazon.com/media-services/).

Design playback authorization, signed access where appropriate, DRM integration, content rights, retention, and regional restrictions. Protect ingest credentials and prevent a public origin from bypassing intended controls. Watch startup delay, rebuffering, rendition changes, and successful playback rather than only encoder uptime.

**Domain exercise:** transcode a short self-created clip into a small output set. Verify audio, captions if used, duration, and playback. Use a local player or tightly scoped distribution. Delete output objects, versions, temporary distributions, jobs/queues where appropriate, and logs. Live channels and rendering workers can continue charging while waiting, so do not leave a live pipeline running for a learning exercise.

## 4. End-user computing and application streaming

Begin with user personas: persistent office desktop, temporary application access, graphics workstation, contractor access, or controlled browser use. Determine application licensing, device support, identity provider, directory needs, storage, latency, peripheral requirements, and data-loss-prevention expectations.

Evaluate Amazon WorkSpaces for appropriate managed desktop scenarios and **Amazon WorkSpaces Applications**, formerly AppStream 2.0, for application streaming. Understand image creation, application updates, user entitlement, session isolation, profiles, scaling, and persistence. [WorkSpaces Applications documentation](https://docs.aws.amazon.com/appstream2/latest/developerguide/what-is-appstream.html).

Do not treat all WorkSpaces variants as equivalent. WorkSpaces Secure Browser has a maintenance notice, while Pools, Thin Client, and PCoIP-related offerings have relevant lifecycle notices as of this review. Select a supported current delivery approach and protocol for a new project. [Maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html), [sunset register](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html).

A remote desktop does not automatically prevent data exfiltration: clipboard, file transfer, browser uploads, screenshots, local devices, and application permissions need explicit consideration. Benchmark actual user locations and network quality. A desktop that launches successfully can still be unusable because of latency or an incompatible peripheral.

**Design exercise:** compare persistent desktop delivery with streamed application access for ten fictional users. Produce an identity flow, application image update process, data persistence policy, latency test plan, and monthly cost model. Keep it design-only unless desktop, directory, license, and fleet costs are understood; if deployed, remove fleets, images, directories, volumes, and other dedicated dependencies when appropriate.

## 5. Contact centers and customer communications

Learn Amazon Connect concepts: contact flows, routing, queues, agent availability, hours of operation, customer context, recording, monitoring, and integration with business systems. Distinguish customer-facing bot behavior from agent assistance, reporting, and case handling.

Study telephony reachability, number availability, supported countries, contact continuity, call quality, and failure routing. Data retention and recording access require intentional design. Use synthetic customer information in exercises and obtain the necessary organizational and legal decisions before production recording or messaging.

For outbound communication, distinguish Amazon SES email delivery from application events and notifications, and investigate AWS End User Messaging for supported messaging channels. Sending requires recipient eligibility, consent handling, identity configuration, reputation management, bounce/complaint handling, and country-specific rules. A successful API response does not prove final customer delivery.

Amazon Pinpoint has an announced October 30, 2026 end-of-support date; avoid building a new learning project around its retiring engagement experience. Do not assume this means every separately branded messaging capability has the same lifecycle. [AWS sunset register](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html).

**Design exercise:** map a customer support journey with authentication, queueing, an agent handoff, a callback alternative, and a CRM lookup. Verify the failure paths for unavailable agents and a slow CRM. Use a simulator or diagram first; phone numbers, calls, recordings, analytics, and outbound messages can incur ongoing charges. No exercise requires contacting real customers.

## 6. Business integration and industry workflows

Learn integration ownership and data contracts before connecting software. Amazon AppFlow can suit supported SaaS data flows; EventBridge and APIs suit event-driven integration; Transfer Family supports managed file-transfer workflows; AWS B2B Data Interchange addresses supported electronic data interchange transformations. [AWS B2B Data Interchange](https://aws.amazon.com/b2b-data-interchange/).

For each flow, document the system of record, identity, schema, schedule, retries, duplicates, reconciliation, audit trail, and deletion behavior. A nightly file transfer needs completeness checks and business acknowledgment, not just a “file uploaded” signal. Schema drift and revoked SaaS credentials are normal operational events to test.

Study specialist domains selectively:

- **Healthcare and life sciences:** investigate HealthLake, HealthImaging, and suitable HealthOmics capabilities alongside standards such as FHIR and DICOM. Learn de-identification, access purpose, data provenance, retention, and workflow reproducibility. Service eligibility does not automatically make a complete application compliant. HealthOmics variant/annotation stores have a separate lifecycle notice; distinguish them from other capabilities.
- **Manufacturing and automotive:** learn asset hierarchies, equipment data quality, disconnected sites, production safety, and operational-technology segmentation. Treat vehicle or industrial data collection as a specialized engineering domain rather than a generic telemetry copy.
- **Supply chain:** study product/location identities, event timeliness, forecast inputs, inventory accuracy, and ERP integration. Evaluate current AWS and partner offerings against the actual business process.
- **Financial services:** study auditability, reconciliation, market-data entitlements, retention, and resilience. Domain regulation and contract interpretation require qualified review rather than assuming a cloud feature settles the question.
- **SAP and mainframe:** understand licensing, certified configurations, transactions, batch windows, integrations, and modernization constraints. Inspect current AWS Transform and partner paths; historical Mainframe Modernization experiences have feature-specific availability changes.

Use the [AWS industry pages](https://aws.amazon.com/industries/) and [healthcare and life sciences resources](https://aws.amazon.com/health/) to find domain architectures and supported products. Build a synthetic-data integration with reconciliation before handling sensitive real records.

## 7. HPC, scientific computing, and quantum

Distinguish **tightly coupled HPC**, where nodes communicate frequently, from **high-throughput batch**, where many jobs can run independently. Learn MPI, Slurm, job queues, parallel speedup, checkpointing, data locality, shared filesystems, and accelerator utilization.

Evaluate AWS Batch for suitable batch execution, AWS Parallel Computing Service for managed Slurm-based environments, and AWS ParallelCluster where its management model fits. Consider EC2 instance selection, Elastic Fabric Adapter, placement, FSx for Lustre, S3 staging, and appropriate visualization tools. More nodes may increase communication overhead without improving time to result. [AWS Parallel Computing Service](https://aws.amazon.com/pcs/).

Measure cost per completed scientific result, including failed jobs, preprocessing, data staging, licenses, and idle scheduling capacity. Spot can be useful for checkpointable work but needs interruption handling. Keep a small local baseline to validate numerical correctness before scaling out.

Amazon Braket offers access to quantum development, simulators, and supported quantum hardware. Learn qubits, gates, measurement, shots, noise, and hybrid workflows before comparing devices. Quantum access does not imply a practical advantage for an arbitrary business problem. [Amazon Braket overview](https://docs.aws.amazon.com/braket/latest/developerguide/what-is-braket.html).

**Exercise:** run a small numerical batch locally, vary worker count, and record speedup and output equivalence. For quantum, use a local simulator for a simple circuit before any paid remote execution. If you use AWS, bound task counts, runtime, and device selection; clean up notebooks, clusters, filesystems, compute, and storage.

## 8. Blockchain, games, geospatial, and space

Blockchain study should begin with the trust model: why multiple parties need shared state, who validates updates, how keys are controlled, and whether a conventional database plus audit records would meet the requirement. Learn transaction finality, network fees, key custody, privacy, and recovery independently of a cloud product.

Amazon Managed Blockchain entered sunset with a September 29, 2027 end-of-support date in the current AWS register. QLDB has already ended support. Treat these as lifecycle and migration study topics, not default services for a new roadmap lab. [AWS sunset register](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html), [full shutdown register](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html).

For games, investigate GameLift Servers and GameLift Streams according to whether you need multiplayer server hosting or interactive game streaming. Learn matchmaking, session placement, player latency, state persistence, anti-abuse controls, fleet scaling, and deployment compatibility.

For geospatial work, Amazon Location Service is relevant to supported maps, places, routes, tracking, and geofencing workflows. Learn location accuracy, geospatial indexing, rate limits, provider terms, and sensitive movement-data handling. Use invented coordinates or public nonpersonal locations in exercises.

For satellite workloads, AWS Ground Station connects ground operations with data processing. Learn contacts, orbital scheduling, antenna compatibility, downlink processing, storage, and partner responsibilities. This is a specialized design topic; no satellite contact or hardware purchase is needed to learn the architecture.

## Common pitfalls

- Choosing a specialized service before learning the underlying domain protocol and failure behavior.
- Treating a vendor certification or eligible service as proof that the complete application meets an industry obligation.
- Reusing old tutorials without checking feature-level lifecycle and new-customer access.
- Leaving media channels, desktops, clusters, notebooks, or telecom resources running after an exercise.
- Testing with real device identities, customer records, or recordings when synthetic data would demonstrate the same concept.

## Capstone and mastery checklist

Choose one domain and produce a four-part portfolio artifact: requirements and constraints, a current service/lifecycle comparison, a small implementation or faithful simulation, and an operations/cost/recovery report. Label simulated behavior clearly. Verify both the happy path and one realistic failure, such as device reconnect, corrupted media input, duplicate business document, interrupted batch job, or unavailable external provider.

- [ ] I can explain the domain's unique latency, correctness, identity, and data requirements.
- [ ] I have checked the exact feature's lifecycle and regional availability.
- [ ] I can compare a managed AWS option with a credible alternative.
- [ ] I can verify a business outcome, not merely a successful resource deployment.
- [ ] I have bounded costs and documented all cleanup dependencies.
- [ ] I know which decisions require domain specialists or existing organizational policy.

## Additional official references

- [AWS IoT documentation](https://docs.aws.amazon.com/iot/)
- [Amazon Connect documentation](https://docs.aws.amazon.com/connect/)
- [Amazon WorkSpaces documentation](https://docs.aws.amazon.com/workspaces/)
- [AWS Batch documentation](https://docs.aws.amazon.com/batch/)
- [Amazon GameLift documentation](https://docs.aws.amazon.com/gamelift/)
- [Amazon Location Service documentation](https://docs.aws.amazon.com/location/)
- [AWS Ground Station documentation](https://docs.aws.amazon.com/ground-station/)

Return to the [roadmap index](README.md) and choose projects and a specialization aligned with the work you want to do.
