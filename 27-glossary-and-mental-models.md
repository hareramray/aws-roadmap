# 27 — Glossary and mental models

[Roadmap index](README.md)

This glossary explains the vocabulary used throughout the roadmap. Service-specific limits and exceptions belong in the linked technical chapters and current official documentation.

## Cloud and architecture

| Term | Meaning |
|---|---|
| Account | An AWS resource/identity/billing boundary; organizations can centrally govern several accounts |
| Region | A geographic service deployment area containing multiple availability zones |
| Availability Zone (AZ) | An isolated infrastructure location within a region; design for failures beyond a single machine |
| Edge location | Infrastructure serving edge functions such as content delivery; not interchangeable with a full region |
| Control plane | APIs and processes that create/configure a service's resources |
| Data plane | The path serving the workload's actual requests or data |
| Shared responsibility | Division of security/operational duties between AWS and the customer, varying by service |
| Managed service | AWS operates some infrastructure/service layers; customers still configure and use it correctly |
| Serverless | Infrastructure management is abstracted and scaling/billing follow service-specific models; servers still exist |
| Elasticity | Adjusting capacity as demand changes |
| Scalability | Ability to handle increased work through a suitable architecture and resources |
| Fault domain | A group of components that can fail together |
| Blast radius | The scope of damage or disruption caused by a failure or compromised privilege |
| Multi-tenancy | Serving multiple customers or groups while enforcing required isolation |
| ADR | Architecture decision record: context, options, decision, consequences, and review trigger |

**Mental model:** draw three maps of a system: the request/data path, the permission/trust path, and the failure/recovery path. A diagram that shows only service logos hides most of the engineering work.

## Identity and security

| Term | Meaning |
|---|---|
| Principal | An identity making a request, such as an assumed role session or service principal |
| ARN | Amazon Resource Name identifying a resource or principal in a service-specific format |
| Authentication | Establishing the caller's identity |
| Authorization | Deciding which action that caller may perform on which resource in what context |
| IAM role | An assumable identity with trust rules and permissions, normally used through temporary sessions |
| STS | Security Token Service, used for temporary security credentials and related identity operations |
| Trust policy | A role policy describing who may assume it and under which conditions |
| Identity policy | Permissions attached to an IAM identity |
| Resource policy | Permissions attached to a supported resource, such as a bucket or key |
| Permissions boundary | A maximum permission boundary for an identity; does not grant permissions by itself |
| Session policy | An additional restriction applied to a role/federated session where supported |
| SCP | Service control policy limiting available permissions in its applicable organization scope |
| RCP | Resource control policy limiting permissions for supported resources in its applicable scope |
| Federation | Trusting an external identity provider to establish sessions/access |
| OIDC / SAML | Standards used in identity federation; exact token/assertion validation matters |
| ABAC | Attribute-based access control using attributes such as tags and session context |
| RBAC | Role-based access control using defined roles and their permissions |
| KMS key | A managed cryptographic key resource with key policy and usage controls |
| Envelope encryption | Encrypt data with a data key, then protect that data key with another key |
| Secret | Sensitive authentication material such as a password or token; an identifier alone is not necessarily secret |
| TLS | Protocol protecting data in transit and authenticating peers according to its configuration |
| MFA | Additional authentication factor beyond a password or another single factor |
| Least privilege | Granting the actions/resources/context needed for a legitimate purpose, then maintaining that fit |

**Mental model:** successful access usually requires a valid identity, applicable permission, reachable endpoint, and the correct resource state. Fixing one does not prove the others. Policy evaluation has special cases; do not reduce all identity and resource policies to one oversimplified formula.

## Networking

| Term | Meaning |
|---|---|
| VPC | Logically isolated virtual network in a region |
| CIDR | Address range notation, such as an IPv4 prefix with a mask length |
| Subnet | An address range within a VPC, placed in an AZ |
| Route table | Rules choosing destinations/next hops for traffic |
| Internet gateway | VPC component supporting internet connectivity with appropriate routes/addressing |
| NAT | Address translation; a typical outbound IPv4 design maps private-source traffic to routable addresses |
| Egress-only internet gateway | IPv6 outbound internet connectivity control that blocks unsolicited inbound initiation |
| Security group | Stateful traffic policy associated with supported resources/network interfaces |
| NACL | Stateless subnet-level network access control list |
| ENI | Elastic network interface carrying addresses and network attributes |
| VPC endpoint | Private connectivity mechanism for supported service access; gateway and interface types differ |
| PrivateLink | Private service connectivity built around supported endpoint/service patterns |
| Peering | Direct private connectivity between two VPCs, subject to routing and feature constraints |
| Transit Gateway | Regional transit connectivity hub for supported network attachments |
| VPN | Encrypted network tunnel; routing/redundancy still need design |
| Direct Connect | Dedicated connectivity option to AWS; privacy of a path and encryption are separate properties |
| DNS | Distributed name-resolution system |
| TTL | Time-to-live; meaning depends on DNS, cache, message, or database context |
| Load balancer | Distributes traffic among healthy destinations using service-specific protocols/policies |
| CDN | Content delivery network that caches/serves content near users according to configured rules |

**Mental model:** debug both directions of a connection. The forward packet path can work while a return path, stateless rule, DNS answer, or application listener breaks the request.

## Data and messaging

| Term | Meaning |
|---|---|
| Object storage | Stores objects addressed by keys and accessed through APIs |
| Block storage | Exposes storage blocks to a host/filesystem or application |
| File storage | Exposes filesystem semantics through supported protocols |
| Durability | Probability/ability of retaining data under the system's stated failure assumptions |
| Availability | Ability to serve requests when needed under defined conditions |
| Replication | Copying changes to other storage/nodes; not automatically protection from malicious or accidental changes |
| Snapshot | Point-in-time representation used for recovery or cloning; implementation varies |
| RPO | Recovery point objective: acceptable data loss expressed as time |
| RTO | Recovery time objective: acceptable time to restore a defined service |
| Transaction | Group of operations with defined atomicity/isolation and durability behavior |
| Partition/shard | A division of data/work used to distribute storage or processing |
| Hot key | Key or partition receiving disproportionate traffic |
| Secondary index | Alternate lookup structure maintained alongside primary data |
| Cache | Faster access layer with explicit freshness/invalidations and service-specific persistence behavior |
| Idempotency | Repeating a logical operation has the intended single-operation effect |
| At-least-once | Delivery/processing can occur more than once; consumers must handle duplicates as needed |
| DLQ | Dead-letter queue holding messages that could not be processed/delivered under the configured policy |
| Visibility timeout | Period during which a received SQS message is normally hidden from other consumers; it is not an exclusive-processing guarantee |
| Backpressure | Signaling or limiting upstream work when downstream capacity is insufficient |
| Saga | Multi-step workflow using local transactions and compensating actions |
| Outbox | Pattern writing business state and an event record atomically before asynchronous publication |
| CDC | Change data capture: extracting database changes for replication/integration |
| OLTP / OLAP | Transaction-oriented operations versus analytical processing |
| Lakehouse | Architecture combining data-lake storage/open formats with table/transaction/governance capabilities |
| Watermark | Stream-processing estimate/policy for how far event-time progress has advanced |

**Mental model:** “stored,” “replicated,” “backed up,” “retained immutably,” and “restorable” are different claims. Ask what failure each mechanism protects against and verify recovery.

## Delivery and operations

| Term | Meaning |
|---|---|
| IaC | Infrastructure as code: versioned declarations/programs managing resources |
| Desired state | Intended configuration against which actual state is compared/reconciled |
| Drift | Difference between expected managed configuration and actual configuration |
| CI | Continuous integration: frequent integration with automated validation |
| CD | Continuous delivery or deployment; define whether production release is automatic |
| Artifact | Versioned output such as an image, package, or compiled bundle |
| Immutable deployment | Replacing an artifact/resource rather than mutating it unpredictably in place |
| Blue/green | Parallel old/new environments with controlled traffic transition |
| Canary | Exposing a change to a limited fraction of requests/users before expanding |
| Feature flag | Runtime control of behavior independent of some deployment changes |
| Metric | Numerical measurement over time |
| Log | Recorded event/context, preferably structured for analysis |
| Trace | Connected spans describing a request's path across components |
| SLI | Service-level indicator measuring a service property |
| SLO | Target for an SLI over a defined window |
| SLA | Agreement with defined service commitments and consequences; not identical to an internal SLO |
| Error budget | Allowed unreliability implied by an SLO over its window |
| Runbook | Actionable operating instructions with prerequisites, checks, and escalation |
| Playbook | Broader decision guidance for a class of situations; terminology varies by team |
| Post-incident review | Evidence-based analysis of impact, causes, response, and improvements |
| FinOps | Collaborative practice of making cloud spending visible, accountable, and valuable |
| Unit cost | Cost per meaningful output, such as a completed job or tenant-month |

## AI and ML

| Term | Meaning |
|---|---|
| Training | Updating model parameters from data |
| Inference | Applying a trained model to input |
| Foundation model | Broadly pretrained model adapted or prompted for many tasks |
| Embedding | Numeric representation intended to capture useful relationships in data |
| Vector retrieval | Finding items by similarity in an embedding space, subject to filters/index behavior |
| RAG | Retrieval-augmented generation: using retrieved context during generation |
| Fine-tuning | Further training a pretrained model on selected data/tasks |
| Groundedness | Degree to which an answer is supported by supplied/allowed evidence |
| Hallucination | Output that presents unsupported or incorrect information as if valid |
| Prompt injection | Untrusted content attempting to redirect model behavior or tool use |
| Evaluation set | Representative cases with defined criteria for judging system behavior |
| Data leakage | Information improperly crossing training/evaluation or access boundaries; specify the context |
| Drift | Change in inputs, target relationships, or behavior over time |
| Agent | Application using model-driven steps/tools; authority must still be enforced outside model text |

## Distinctions worth remembering

- Multi-AZ is not the same as multi-region, and neither automatically proves recovery.
- Encryption does not repair an overly broad permission policy.
- A private subnet does not automatically make every application interaction secure.
- A cache, replica, and backup solve different problems.
- An event being accepted does not prove its business workflow completed.
- A service quota is not a performance guarantee; available regional capacity is another concern.
- “Serverless,” “managed,” and “free allowance” do not mean zero operational or financial responsibility.
- A successful infrastructure deployment does not prove application correctness.
- A foundation model's confidence is not evidence of correctness or authorization.

For formal definitions and service-specific usage, consult [AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/Welcome.html) and the official links in the relevant chapter.
