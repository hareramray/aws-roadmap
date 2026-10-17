# 09 — Security engineering, detection, encryption, and compliance

[Roadmap index](README.md) · Prerequisites: [IAM](04-iam-and-identity.md), [networking](05-networking-and-dns.md), [compute](06-compute-and-load-balancing.md), [storage](07-storage-and-backup.md)

Reviewed: **2026-10-18**. Security is a continuous engineering process: identify assets and trust boundaries, prevent unnecessary access, detect misuse, respond effectively, and demonstrate recovery. Enabling a collection of services is only one part of that process.

## Outcomes

Threat-model a workload, select appropriate controls, protect encryption keys and secrets, establish useful security telemetry, distinguish major detection services, write an incident playbook, and map technical evidence to a compliance requirement.

## 1. Start with assets and threats

List the assets: customer data, account access, source code, deployment authority, secrets, backups, logs, domain names, and availability. Identify entry points such as public APIs, uploads, CI jobs, administrator access, dependencies, and third-party integrations.

Draw trust boundaries across browser, API, application, database, account, external provider, and administrative plane. For each boundary ask what can be forged, exposed, altered, replayed, or exhausted. Prioritize threats using likelihood, impact, exposed privileges, and ability to detect/recover.

A practical threat register has an owner, scenario, affected assets, existing prevention, detection evidence, response action, residual risk, and review date. Avoid a checklist that says merely “encryption enabled”; specify who can decrypt and which threat encryption addresses.

Use defense in depth: strong identity, least privilege, private service paths where appropriate, application authorization, validated inputs, encryption, logging, backups, and controlled deployments. Each layer should have a defined purpose and a test.

## 2. Identity and organization controls

Apply [module 04](04-iam-and-identity.md): federated human access, MFA, temporary workload credentials, narrow trust, separate runtime/deployment roles, and limited `PassRole`. Protect root recovery and emergency access, and test the emergency procedure without making it everyday practice.

For an established organization, account separation, organizational units, SCPs/RCPs, delegated security administration, and centralized log archives can limit exposure. Begin guardrails in a test OU; a broad deny can break essential automation, incident access, or service-linked operations. Keep the management account free of ordinary workloads and grant its access sparingly.

A data perimeter combines identity, resource, and network conditions. It should answer which principals can access which data from which contexts, including legitimate service-to-service access. A private endpoint by itself does not prove that a bucket cannot be accessed through another permitted path.

## 3. Encryption and key management

Distinguish encryption at rest, encryption in transit, and encryption performed by the application before data reaches a service. Encryption protects particular exposures; it does not stop an authorized compromised identity from requesting plaintext.

AWS KMS manages cryptographic keys and integrates with many services. Study symmetric/asymmetric keys, key policies, IAM policies, grants, aliases, encryption context, rotation, imported material, multi-Region keys, and separation of key administrators from key users. An alias is a pointer, not the key itself.

Envelope encryption uses a data key for the data and a higher-level key to protect that data key. Rotation of KMS key material does not automatically re-encrypt all previously stored application data. Retain the ability to decrypt old data according to its retention requirements and use service-supported re-encryption workflows when necessary. [KMS FAQ](https://aws.amazon.com/kms/faqs/)

Key deletion can make encrypted data unrecoverable. A retained snapshot is useless if the necessary decryption key is gone. Treat disabling and scheduling deletion as changes requiring dependency inventory and an intentional recovery window; never delete a shared key as casual lab cleanup. [Deleting KMS keys](https://docs.aws.amazon.com/kms/latest/developerguide/deleting-keys.html)

CloudHSM is relevant when dedicated HSM control and specialized cryptographic requirements justify its operations. ACM manages supported certificate workflows; study validation, renewal, supported endpoints, certificate placement, and imported-certificate responsibilities. Private certificate authorities add issuance, trust, revocation, and cost considerations.

## 4. Secrets and application security

Secrets Manager supports storing and rotating secrets with appropriate integrations. Parameter Store can hold configuration and supported encrypted values, with different features and pricing. Select based on rotation, lifecycle, access, change notification, and application needs rather than assuming the services are identical. [Secrets Manager overview](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

Fetch secrets with a runtime role; cache them for a bounded interval and handle rotation without unnecessary outages. Never expose secrets in source control, container layers, AMIs, bootstrap scripts, telemetry, exception messages, or screenshots. Rotation needs to update both the secret and the target system using a tested sequence.

Application controls remain essential: validate authorization on each object/action, use parameterized queries, constrain uploads, apply secure session handling, validate tokens, set appropriate browser security headers, and patch dependencies. A WAF does not correct broken object authorization or replace secure code.

Build software supply-chain controls into CI: dependency and image scanning, trusted sources, version pinning where appropriate, artifact provenance/signing, isolated build permissions, protected release branches, and reviewed infrastructure changes. A clean vulnerability scan is evidence about covered checks at a point in time, not proof of invulnerability.

## 5. Network and edge protection

Use security groups for workload communication, NACLs for applicable subnet-level guardrails, and controlled routing/inspection where justified. Protect IPv6 paths alongside IPv4. Private databases still need database authorization, TLS, and patching.

AWS WAF filters supported application requests using web ACLs and rules. Learn managed rule groups, custom rules, rate-based controls, rule ordering, scope, labels, logging, and count-mode evaluation before blocking production traffic. Tune using legitimate request samples; aggressive rules can become a self-inflicted outage. Use the current AWS WAF APIs and guides rather than WAF Classic tutorials.

AWS Shield addresses DDoS protection with different service tiers and engagement features. Network Firewall, Firewall Manager, and Route 53 Resolver DNS Firewall address different control and administration needs. Assess integration, coverage, logs, and cost for the actual topology rather than expecting one firewall product to inspect every path.

## 6. Detection and investigation services

| Service/capability | Main question it helps answer | What it does not establish alone |
| --- | --- | --- |
| CloudTrail | Which recorded AWS API action occurred, by whom, and when? | Every application event or every data access by default |
| AWS Config | What supported configuration existed and how does it compare with rules? | Whether every runtime action was malicious |
| GuardDuty | Does supported telemetry indicate suspicious or malicious activity? | Automatic prevention of every threat |
| Inspector | What covered vulnerabilities/exposures affect supported workloads? | Complete business-logic security |
| Macie | What sensitive data is discovered in supported S3 analysis? | Classification of every byte in every service |
| Security Hub CSPM | How do controls/findings describe posture across enabled scope? | Legal compliance certification |
| Security Hub capabilities | How can security findings and coverage be prioritized/correlated? | A substitute for enabling/configuring the underlying coverage |
| Detective | What investigation context connects suspicious activity? | A replacement for an incident investigator |
| Security Lake | How can supported security data be centralized for analysis? | Automatically complete detections and response playbooks |

Security Hub CSPM evaluates posture and aggregates supported findings. Its controls often depend on AWS Config and correct recording scope. Central aggregation does not retroactively enable all Regions or all underlying security features. [Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html), [Security Hub coverage findings](https://docs.aws.amazon.com/securityhub/latest/userguide/coverage-findings.html)

GuardDuty's foundational and optional protection capabilities have different coverage and pricing. Understand the relevant data sources, resource enrollment, findings, suppression, and export paths. Treat a finding as evidence to investigate, with severity and context; not every finding is a confirmed compromise.

## 7. Logging that can support response

CloudTrail Event history is useful for recent Regional management activity. Trails deliver selected events to durable destinations such as S3, with other integrations available. Explicitly choose management/data event coverage, regions, retention, encryption, validation, and access. High-volume data events can be valuable and costly. [CloudTrail overview](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)

Centralize important logs in a boundary ordinary workloads cannot modify. Test delivery and alerts for interruption. Keep timestamps, request IDs, principal/session identity, resource identifiers, and relevant application context. Avoid logging full tokens, payment details, secrets, or unnecessary personal data.

For new learning deployments, use supported CloudTrail trails and suitable analysis paths such as S3/Athena. **CloudTrail Lake and AWS Audit Manager are listed in maintenance as of this review; new customers cannot onboard to maintenance-stage services/features.** Existing deployments require their own lifecycle review. [AWS services in maintenance](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html)

## 8. Incident response and recovery

Prepare playbooks for compromised credentials, exposed data, suspicious compute, malicious dependency, ransomware-like deletion, and denial of service. Each playbook needs a trigger, owner, evidence sources, containment choices, approval authority within the organization, recovery steps, and communication path.

Containment must match the resource: revoke or restrict identity sessions, isolate workloads, block malicious paths, and preserve necessary evidence. Deleting a suspicious instance immediately may destroy evidence. Isolation may also disrupt collection; plan a forensics access path and chain of custody before an incident.

After containment, remove the cause, rotate affected secrets, rebuild trusted artifacts, restore verified data if necessary, and monitor for recurrence. Measure time to detect, triage, contain, and recover. A blameless review should produce specific control and runbook improvements with owners.

Automation begins with enrichment and notification, then carefully bounded remediation. Use idempotency, retries, dead-letter handling, scope checks, audit logs, and rollback. Do not let a sample finding or broad rule automatically terminate unrelated workloads.

## 9. Compliance as evidence and scope

Define the applicable requirements with the responsible organization, then map controls to evidence and owners. AWS Artifact can provide AWS reports and agreements relevant to inherited controls; your application configuration, procedures, data handling, and evidence remain separate responsibilities.

Document data inventory, residency, retention/deletion, access review, change control, vulnerability remediation, backup verification, and incident handling. A dashboard score is not an audit conclusion. Evaluate scope carefully: an excluded account, Region, resource type, or unrecorded setting can create a misleadingly clean view.

## Lab A — Threat model and incident tabletop

**Type:** local design exercise; no deployment or charge.

Use the upload service from [module 03](03-cloud-foundations.md). Model a stolen CI credential, an unauthorized file download, and accidental deletion. For each scenario create one preventive control, one detection with required evidence, one containment step, and one recovery test. Add a responsibility matrix for application, platform, security, and account ownership.

**Verify:** a reviewer can follow the response without guessing account, resource, identity, or owner. Explain how evidence survives the compromised workload and how a false positive is handled. **Cleanup:** use fabricated identifiers and remove any accidentally included confidential data from the exercise notes.

## Lab B — Sample finding triage

**Type:** optional sandbox security-service exercise. Review costs and existing organization controls first. If GuardDuty is already managed centrally, work within the assigned sandbox and do not change its organization configuration.

1. Use an existing authorized GuardDuty deployment, or enable it deliberately for one dedicated sandbox Region after checking the selected features and pricing.
2. Generate supported **sample findings** from the console. These contain placeholder values and are marked as samples; they do not require launching an attack.
3. Inspect one finding's type, severity, resource, time, account, Region, and sample marker. Write a triage note that separates observed sample fields from an investigator's hypotheses.
4. Optionally route sample findings to a dedicated EventBridge target/log group for inspection. Keep automatic resource-remediation targets disabled.
5. Confirm that the event reaches the intended destination and can be traced back to its finding.

**Verify:** the sample is identifiable, the routing rule is scoped correctly, and the proposed playbook names the additional evidence needed for a real incident. Samples test processing and triage; they do not prove end-to-end threat detection. [Generate GuardDuty sample findings](https://docs.aws.amazon.com/guardduty/latest/ug/sample_findings.html)

**Cleanup/cost:** archive only sample findings, remove lab event targets/rules/logs, and disable only protections you enabled solely for a disposable lab when that matches the sandbox policy. Preserve any pre-existing organization security baseline. Security-service analysis, logs, exports, and integrated services can incur charges independently.

## Mastery checklist

- [ ] Map an actual threat to a prevention, detection, and recovery control.
- [ ] Explain key access, rotation, and recovery dependencies.
- [ ] Distinguish CloudTrail, Config, GuardDuty, Inspector, Macie, and posture management.
- [ ] Identify Region/account/feature gaps in monitoring coverage.
- [ ] Triage a finding and preserve evidence before destructive action.
- [ ] Produce an incident playbook and control evidence with clear ownership.

Further references: [AWS security incident response guide](https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html), [AWS KMS guide](https://docs.aws.amazon.com/kms/latest/developerguide/), [AWS WAF guide](https://docs.aws.amazon.com/waf/latest/developerguide/), [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/), [AWS Artifact guide](https://docs.aws.amazon.com/artifact/latest/ug/).
