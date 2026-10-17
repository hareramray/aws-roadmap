# 30 — AWS service coverage and specialization map

[Roadmap index](README.md) · Reviewed: **2026-10-18**

This map connects the major AWS service families to the roadmap. Use it to find gaps and choose deeper study, not to deploy every product. The [official service categories](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/amazon-web-services-cloud-platform.html) supply the broad taxonomy; individual product documentation and lifecycle notices determine what is usable for your workload.

The map is not an exhaustive inventory of APIs, features, Regions, instance types, engines, partner products, or launch announcements. A linked chapter teaches the relevant discipline; it may introduce a specialist product without providing a complete implementation tutorial.

## How to read the depth and status

- **Core:** explain the model, implement a small example, diagnose a failure, and clean it up. These skills support most AWS roles.
- **Build:** implement when relevant to your role or project, including security, deployment, telemetry, and recovery.
- **Specialist:** understand the use case and alternatives, then undertake domain-specific study before production use. Design exercises can be sufficient during the general roadmap.
- **Lifecycle:** learn recognition, assessment, and migration. A legacy name or maintenance product is not a required new deployment.

Unmarked products are **candidates to evaluate**, not a promise of new-customer access, feature support, pricing, or availability in your account. Check those properties immediately before use. Maintenance, sunset, and shutdown are different states; see the lifecycle section below.

## Foundations, compute, and application execution

| Area and service families | Depth | Curriculum | Required learning evidence |
| --- | --- | --- | --- |
| Accounts, Regions, AZs, edge locations, service endpoints and quotas | Core | [03 Foundations](03-cloud-foundations.md), [02 Setup](02-account-and-tooling.md) | Scope diagram, identity verification, quota and cost assumptions |
| EC2, AMIs, launch templates, Auto Scaling, Elastic Load Balancing | Core | [06 Compute](06-compute-and-load-balancing.md) | Replace an instance, explain health checks and scaling behavior |
| Lambda and event-source integrations | Build | [10 Serverless](10-serverless-and-integration.md) | Idempotent processing, bounded concurrency, retry and failure evidence |
| ECS, Fargate, ECR, ECS Service Connect | Build | [11 Containers](11-containers-and-kubernetes.md) | Immutable release, workload identity, networking, rollback |
| EKS, workload identity, autoscaling and hybrid deployment options | Build | [11 Containers](11-containers-and-kubernetes.md), [21 Hybrid](21-migration-hybrid-and-edge.md) | Cluster/workload responsibility matrix and upgrade procedure |
| Elastic Beanstalk, Lightsail, application hosting choices | Build | [06 Compute](06-compute-and-load-balancing.md), [13 Delivery](13-devops-and-delivery.md) | Compare abstraction, customization, observability, and migration effort |
| AWS Batch, accelerated EC2, Graviton, placement and EFA | Specialist | [06 Compute](06-compute-and-load-balancing.md), [22 Specialties](22-specialized-domains.md) | Benchmark useful work, checkpoint interruption, explain hardware compatibility |

## Networking, identity, and protection

| Area and service families | Depth | Curriculum | Required learning evidence |
| --- | --- | --- | --- |
| VPC, subnets, routing, NAT, IPv6, security groups, NACLs, IPAM | Core | [05 Networking](05-networking-and-dns.md) | Trace forward/return paths and address-family-specific controls |
| Route 53, Resolver, DNS Firewall, Cloud Map | Core/Build | [05 Networking](05-networking-and-dns.md) | DNS ownership, resolution path, caching, discovery and failure tests |
| CloudFront, Global Accelerator, ALB/NLB/GWLB | Build | [05 Networking](05-networking-and-dns.md), [06 Compute](06-compute-and-load-balancing.md) | Compare caching, routing, protocol and inspection responsibilities |
| PrivateLink, VPC endpoints, peering, Transit Gateway, Cloud WAN | Build | [05 Networking](05-networking-and-dns.md), [21 Hybrid](21-migration-hybrid-and-edge.md) | Route/authorization matrix and segmentation verification |
| Direct Connect, Site-to-Site VPN, Client VPN | Build | [21 Hybrid](21-migration-hybrid-and-edge.md) | Redundancy, encryption assumptions, BGP/DNS and failover procedure |
| VPC Lattice and Verified Access | Specialist | [05 Networking](05-networking-and-dns.md), [09 Security](09-security-and-compliance.md) | Distinguish application connectivity from user/device-aware access policy |
| IAM, STS, Identity Center, IAM Access Analyzer | Core | [04 Identity](04-iam-and-identity.md) | Temporary sessions, trust policy, least privilege, denied-case tests |
| Cognito, Verified Permissions, Directory Service and IAM Roles Anywhere | Build/Specialist | [04 Identity](04-iam-and-identity.md), [09 Security](09-security-and-compliance.md) | Separate customer login, application authorization, directory, external workload identity |
| KMS, Secrets Manager, Parameter Store, ACM, Private CA, CloudHSM | Core/Build | [09 Security](09-security-and-compliance.md) | Key/secret/certificate lifecycle, recovery dependency, access separation |
| WAF, Shield, Network Firewall, Firewall Manager | Build | [09 Security](09-security-and-compliance.md) | Coverage map, false-positive testing, alert and change ownership |
| GuardDuty, Inspector, Macie, Detective, Security Hub/CSPM, Security Lake | Build | [09 Security](09-security-and-compliance.md), [16 Operations](16-observability-and-operations.md) | Explain threat, vulnerability, classification, posture and investigation differences |
| Artifact, compliance evidence and security reference architectures | Build | [09 Security](09-security-and-compliance.md), [19 Governance](19-organizations-and-governance.md) | Map requirements to inherited controls and customer evidence |

For the less familiar networking branches, start with [VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/what-is-vpc-lattice.html) and [Verified Access](https://docs.aws.amazon.com/verified-access/latest/ug/what-is-verified-access.html), then compare them with existing network and identity controls.

## Storage, transactional data, and analytics

| Area and service families | Depth | Curriculum | Required learning evidence |
| --- | --- | --- | --- |
| S3 general purpose buckets, storage classes, access points, replication, Object Lock | Core | [07 Storage](07-storage-and-backup.md) | Private access, version recovery, lifecycle and complete cleanup |
| S3 Express One Zone and directory buckets | Specialist | [07 Storage](07-storage-and-backup.md) | Zonal scope, endpoint/auth differences and workload benchmark |
| S3 Tables and S3 Vectors | Build/Specialist | [14 Analytics](14-data-engineering-and-analytics.md), [15 AI](15-ai-ml-and-generative-ai.md), [07 Storage](07-storage-and-backup.md) | Separate table/vector buckets, query semantics, integration and access controls |
| EBS, snapshots, instance store | Core | [07 Storage](07-storage-and-backup.md), [06 Compute](06-compute-and-load-balancing.md) | Persistent versus ephemeral state and verified restore |
| EFS; FSx for Windows File Server, Lustre, ONTAP, OpenZFS | Build/Specialist | [07 Storage](07-storage-and-backup.md), [22 Specialties](22-specialized-domains.md) | Protocol compatibility, permissions, performance and backup requirements |
| AWS Backup, backup vaults, recovery points, restore testing | Core/Build | [07 Storage](07-storage-and-backup.md), [17 Reliability](17-reliability-and-disaster-recovery.md) | Application recovery and encryption-key dependency evidence |
| RDS engines, Aurora, RDS Proxy, replicas and Multi-AZ variants | Core/Build | [08 Databases](08-databases-and-caching.md) | Query plan, connection behavior, availability versus read scaling |
| Aurora DSQL | Specialist | [08 Databases](08-databases-and-caching.md), [18 Architecture](18-architecture-and-system-design.md) | SQL compatibility, transaction conflicts, regional topology and latency |
| DynamoDB, indexes, Streams, DAX, global tables | Core/Build | [08 Databases](08-databases-and-caching.md), [10 Integration](10-serverless-and-integration.md) | Key design, conditions, consistency mode, hot-partition diagnosis |
| ElastiCache, Valkey/Redis OSS/Memcached, MemoryDB | Build | [08 Databases](08-databases-and-caching.md) | Eviction, stale data, durable versus cache configuration, failure fallback |
| DocumentDB, Neptune, Keyspaces, Timestream for InfluxDB | Specialist | [08 Databases](08-databases-and-caching.md), [22 Specialties](22-specialized-domains.md) | Document, graph, wide-column or time-series fit and exact compatibility |
| Glue, Data Catalog, Lake Formation, EMR, Athena | Build | [14 Analytics](14-data-engineering-and-analytics.md) | Catalog, file/table formats, partitioning, governance and scan cost |
| Kinesis Data Streams, Data Firehose, MSK, Managed Service for Apache Flink | Build | [14 Analytics](14-data-engineering-and-analytics.md), [10 Integration](10-serverless-and-integration.md) | Partition/order assumptions, lag, replay and delivery guarantees |
| Redshift, OpenSearch Service, OpenSearch Ingestion | Build/Specialist | [14 Analytics](14-data-engineering-and-analytics.md), [08 Databases](08-databases-and-caching.md) | Warehouse versus search index, ingestion and workload isolation |
| DataZone, Clean Rooms, Data Exchange, Entity Resolution | Specialist | [14 Analytics](14-data-engineering-and-analytics.md), [09 Security](09-security-and-compliance.md) | Discovery, sharing, privacy boundaries, matching quality and permissions |
| Amazon Quick / Quick Sight analytics and business workflows | Build/Specialist | [14 Analytics](14-data-engineering-and-analytics.md), [22 Specialties](22-specialized-domains.md) | Metric definitions, row-level access, sharing and connected-action review |

Newer branches have dedicated models: [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html), [S3 Vectors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html), and [Aurora DSQL](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/what-is-aurora-dsql.html). Study their documentation rather than assuming older S3 or PostgreSQL behavior applies unchanged.

## Integration, software delivery, and AI

| Area and service families | Depth | Curriculum | Required learning evidence |
| --- | --- | --- | --- |
| API Gateway, AppSync, Amplify, Cognito | Build | [10 Integration](10-serverless-and-integration.md), [04 Identity](04-iam-and-identity.md) | API contracts, authorization, frontend deployment and tenant isolation |
| SQS, SNS, EventBridge, Scheduler, Pipes, Step Functions | Core/Build | [10 Integration](10-serverless-and-integration.md) | Duplicate delivery, retries, ordering, workflow failure and redrive |
| Amazon MQ, AppFlow, Transfer Family, B2B Data Interchange | Build/Specialist | [10 Integration](10-serverless-and-integration.md), [21 Migration](21-migration-hybrid-and-edge.md), [22 Specialties](22-specialized-domains.md) | Broker/protocol fit, data contracts and reconciliation |
| CloudFormation, CDK, SAM, Cloud Control API and provider tooling | Build | [12 Infrastructure as code](12-infrastructure-as-code.md) | Repeatable plans, state/ownership, drift and safe rollback |
| CodeCommit, CodeBuild, CodePipeline, CodeDeploy, CodeArtifact, Signer | Build | [13 Delivery](13-devops-and-delivery.md) | Reviewed artifacts, federated pipeline access, testing and release recovery |
| CLI, SDKs, CloudShell, IDE toolkits, Device Farm | Core/Build | [02 Tooling](02-account-and-tooling.md), [29 CLI/SDK](29-cli-and-sdk-reference.md), [13 Delivery](13-devops-and-delivery.md) | Credential-chain diagnosis, pagination, retries and client testing |
| Bedrock model inference, Knowledge Bases, Guardrails and evaluations | Build | [15 AI/ML](15-ai-ml-and-generative-ai.md) | Retrieval quality, safety evaluation, data handling and token/request cost |
| Bedrock AgentCore and agent frameworks | Build/Specialist | [15 AI/ML](15-ai-ml-and-generative-ai.md) | Tool authorization, runtime identity, memory boundaries, trace and evaluation |
| SageMaker AI training, processing, deployment, pipelines and ML infrastructure | Build/Specialist | [15 AI/ML](15-ai-ml-and-generative-ai.md) | Dataset/version lineage, evaluation, serving behavior and lifecycle review |
| Textract, Transcribe, Translate, Polly, Lex, Comprehend, Rekognition, Personalize | Specialist | [15 AI/ML](15-ai-ml-and-generative-ai.md), [22 Specialties](22-specialized-domains.md) | Task accuracy, supported languages/formats, consent, quotas and feature status |

[AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html) is a separate study branch from model inference. Managed runtime capabilities do not remove application authorization or business approval requirements. Older agent tutorials may use maintenance-stage Agents Classic; check the exact product generation.

## Operations, governance, cost, and migration

| Area and service families | Depth | Curriculum | Required learning evidence |
| --- | --- | --- | --- |
| CloudWatch metrics/logs/alarms, Synthetics, RUM; OpenTelemetry and X-Ray paths | Core/Build | [16 Operations](16-observability-and-operations.md) | Correlate symptoms, logs, traces and user-impact signals |
| Managed Service for Prometheus, Managed Grafana | Build | [16 Operations](16-observability-and-operations.md), [11 Containers](11-containers-and-kubernetes.md) | Metric cardinality, collection, dashboard access and alert ownership |
| CloudTrail trails, Config, Systems Manager, AppConfig, AWS Health | Core/Build | [16 Operations](16-observability-and-operations.md), [09 Security](09-security-and-compliance.md) | Audit evidence, configuration history, patching and controlled configuration rollout |
| Organizations, Control Tower, RAM, Service Catalog, Resource Explorer, tagging | Build | [19 Governance](19-organizations-and-governance.md) | Account boundaries, approved provisioning, inventory and shared-resource access |
| Budgets, Cost Explorer, Data Exports, Cost Anomaly Detection, Compute Optimizer | Core/Build | [20 FinOps](20-cost-finops-and-performance.md) | Allocation, forecast, unit cost and measured rightsizing |
| Savings Plans, reservations, Capacity Reservations, License Manager | Build | [20 FinOps](20-cost-finops-and-performance.md), [06 Compute](06-compute-and-load-balancing.md) | Separate discounts, capacity, utilization and licensing obligations |
| Well-Architected Tool, Resilience Hub, Fault Injection Service, ARC | Build | [17 Reliability](17-reliability-and-disaster-recovery.md), [18 Architecture](18-architecture-and-system-design.md) | Review evidence, bounded failure experiments and tested recovery |
| AWS Transform MGN (formerly Application Migration Service), DMS, schema conversion, DataSync, AWS Transform | Build | [21 Migration](21-migration-hybrid-and-edge.md) | Inventory, compatibility, change capture, validation, cutover and rollback |
| Storage Gateway, Outposts, Local Zones, Wavelength, hybrid containers | Specialist | [21 Hybrid/edge](21-migration-hybrid-and-edge.md), [22 Specialties](22-specialized-domains.md) | Local dependencies, connectivity loss, data ownership and supported footprint |
| Support, Trusted Advisor, re:Post, Skill Builder, partners and Marketplace | Core/Build | [28 Resources](28-official-resources-and-updates.md), [24 Career](24-certifications-and-career-paths.md) | Escalation plan, support entitlements, learning evidence and procurement review |

## Specialist domains and less common requirements

Every row below starts at [22 — Specialized domains](22-specialized-domains.md). These are explicit discovery branches, not omitted subjects or mandatory paid labs.

| Domain and candidates | Additional chapter | What deeper study must address |
| --- | --- | --- |
| IoT Core, Device Management, Greengrass V2, SiteWise, TwinMaker | [10 Integration](10-serverless-and-integration.md), [21 Edge](21-migration-hybrid-and-edge.md) | Per-device identity, MQTT, provisioning, reconnect bursts, firmware and physical safety |
| MediaConvert, MediaLive, MediaPackage, MediaConnect, MediaTailor, IVS | [05 Delivery networks](05-networking-and-dns.md) | Ingest, codecs, packaging, rights, playback quality and continuously running costs |
| Deadline Cloud, scientific rendering and virtual workstations | [06 Compute](06-compute-and-load-balancing.md) | Asset movement, software licensing, job correctness and worker utilization |
| WorkSpaces, WorkSpaces Applications and supported remote-display options | [04 Identity](04-iam-and-identity.md) | User experience, image lifecycle, persistence, entitlement and variant-specific lifecycle |
| Connect, SES, End User Messaging and supported Chime SDK capabilities | [10 Integration](10-serverless-and-integration.md) | Contact routing, delivery/reputation, consent, recording access and feature-specific status |
| Business applications, Amazon Quick and SaaS/ERP integration | [14 Analytics](14-data-engineering-and-analytics.md) | Systems of record, connectors, action authorization, reconciliation and business ownership |
| Supply chain and Amazon Connect Decisions; legacy AWS Supply Chain context | [14 Analytics](14-data-engineering-and-analytics.md) | Product/location identities, forecast validation, supplier data and operational decisions |
| HealthLake, HealthImaging, supported HealthOmics capabilities | [09 Security](09-security-and-compliance.md), [15 AI](15-ai-ml-and-generative-ai.md) | FHIR/DICOM, genomics workflows, provenance, access purpose and sensitive-data controls |
| Financial services, SAP, mainframe, automotive, industrial and telecom architectures | [21 Migration](21-migration-hybrid-and-edge.md) | Domain contracts, licensing, specialized protocols and current AWS/partner solutions |
| Parallel Computing Service, ParallelCluster, Batch, EFA, FSx for Lustre | [06 Compute](06-compute-and-load-balancing.md) | MPI/Slurm, scaling efficiency, checkpoints and cost per scientific result |
| Braket and hybrid quantum workflows | [15 AI/numerical workloads](15-ai-ml-and-generative-ai.md) | Circuit correctness, noise, shots, simulator baseline and hardware-task cost |
| Ground Station and satellite data processing | [14 Data pipelines](14-data-engineering-and-analytics.md) | Contact planning, antenna compatibility, downlink and processing dependencies |
| GameLift Servers, GameLift Streams and game backends | [18 Architecture](18-architecture-and-system-design.md) | Session placement, player latency, state, abuse controls and fleet economics |
| Location Service and geospatial applications | [14 Analytics](14-data-engineering-and-analytics.md) | Maps/routes/geofences, accuracy, provider terms and movement-data privacy |
| Blockchain, ledger requirements and Managed Blockchain migration | [08 Databases](08-databases-and-caching.md) | Trust/finality, key custody and whether conventional auditable storage suffices |

AWS describes [Amazon Connect Decisions](https://aws.amazon.com/products/connect/decisions/) as the next generation of AWS Supply Chain. Current naming matters when searching documentation. Use the [industry directory](https://aws.amazon.com/industries/) for additional domain-specific architectures; a service's eligibility does not certify your whole application.

## Lifecycle overlay: recognize old tutorials

The following examples are deliberately excluded from required new builds. They are not a complete retirement register.

| State at review | Examples to recognize | Roadmap treatment |
| --- | --- | --- |
| Maintenance; no new onboarding | App Runner, CodeCatalyst, Cloud9, CloudTrail Lake, Audit Manager, S3 Select/Object Lambda, Timestream for LiveAnalytics | Inspect notices and choose a supported architecture; preserve migration context |
| Feature-level maintenance | Agents Classic; selected SageMaker AI capabilities; Simple AD; SiteWise Monitor; SNS Message Data Protection | Do not transfer the feature's status to the entire parent service |
| Announced sunset | Managed Blockchain, DevOps Guru, WorkMail, selected WorkSpaces offerings, Q Developer IDE plugins | Review exact scope and deadlines; plan migration for existing use |
| Support date already passed | App Mesh, Proton, WAF Classic, Greengrass V1 | Study successor patterns; exclude from new project instructions |
| Full shutdown examples | QLDB, WorkDocs, IoT Analytics/Events, Elastic Transcoder, MediaStore, RoboMaker, FinSpace | Historical recognition only; do not attempt a fresh deployment |

Verify the current [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html), [sunset register](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html), and [shutdown register](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html). Similar names are not interchangeable: Inspector differs from Inspector Classic, AWS Backup differs from retired Backup Storage Service, and S3 Glacier classes differ from legacy Glacier interfaces.

## Discovery protocol for anything missing or newly launched

1. Define the problem without a product name: data, users, latency, reliability, budget and domain constraints.
2. Find the family in this map, then use the [official resource process](28-official-resources-and-updates.md) to locate the current product guide.
3. Check lifecycle, Region/partition, account-plan/new-customer restrictions, engine/runtime support and quotas. Record the exact feature and review date.
4. Write an architecture decision using [engineering templates](32-engineering-templates.md): alternatives, identity, networking, data lifecycle, failure behavior and ownership.
5. Build the smallest meaningful example or a clearly labeled simulation. Test a negative case, collect telemetry, estimate cost and prove cleanup.
6. Add evidence to [the completion checklist](26-master-completion-checklist.md) and connect it to a [portfolio project](23-projects-and-portfolio.md). Update the relevant chapter when findings change its advice.

Completion means you can explain all major families, implement your chosen core/build path, and investigate a specialist responsibly. It does not mean memorizing every AWS product or enabling them all in one account.
