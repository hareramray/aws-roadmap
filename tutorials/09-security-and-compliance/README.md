# Security and compliance tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../09-security-and-compliance.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — Keys, secrets and application protection](01-keys-secrets-and-application-protection.md) | Encryption-context test and secret/certificate lifecycle designs |
| [02 — Detection, evidence and incident response](02-detection-and-incident-response.md) | Sample-finding pipeline and an evidence-preserving playbook |
| [03 — Threat models, network controls and compliance](03-threat-model-network-and-compliance.md) | Threat/control tests, edge-policy design and auditable evidence map |

## Coverage mapping

| Original heading and named subtopics | Tutorials |
| --- | --- |
| Assets/threats/trust boundaries, threat register and defense in depth | 03 |
| Identity/org: federation/MFA/root, roles/PassRole, accounts/OUs/SCP/RCP, delegated administration/data perimeter | 03 |
| Encryption: at rest/transit/client-side, KMS types/policies/grants/aliases/context/rotation/import/multi-Region/deletion, envelope encryption, HSM/ACM/Private CA | 01 |
| Secrets/app: Secrets Manager/Parameter Store, rotation/cache, tokens/uploads/SQL/browser controls, supply chain | 01, 03 |
| Network/edge: SG/NACL/IPv6, WAF current rules/count/rate/labels/logs, Shield, Firewall Manager, Network Firewall/DNS Firewall | 03 |
| Detection/investigation: CloudTrail, Config, GuardDuty, Inspector, Macie, Security Hub/CSPM, Detective/Security Lake | 02 |
| Logging: management/data events/trails/retention/integrity/centralization/privacy/lifecycle | 02 |
| Incident response: credentials/data/compute/dependency/deletion/DoS, containment/forensics/rebuild/restore/automation/review | 02, 03 |
| Compliance: scope, Artifact/inherited controls, data inventory/residency/retention/reviews/evidence | 03 |
| Threat tabletop and sample-finding labs; mastery | 02, 03; verification in all |

Use sample findings and fabricated data. No tutorial requires launching an attack, contacting third parties, changing organization-wide controls, or deploying irreversible retention.
