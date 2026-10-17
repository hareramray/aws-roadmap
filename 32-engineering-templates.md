# 32 — Reusable engineering templates

[Roadmap index](README.md)

Copy the relevant template into your own project documentation. Replace placeholder text with specific evidence; delete sections that truly do not apply and explain significant omissions. Do not store real credentials, customer data, or private account details in a public repository.

## 1. Learning journal

```markdown
# Learning entry: <topic>
Date:
Related roadmap chapter:
Starting question:
What I expected:
What I built/read:
What actually happened:
Evidence (sanitized commands, logs, diagram, test):
Mechanism I can now explain:
Mistake or misconception corrected:
Remaining question:
Resource cleanup status:
Next small experiment:
```

Keep entries short enough to maintain. A useful note explains a mechanism or decision, rather than transcribing a video.

## 2. Lab plan and report

```markdown
# Lab: <name>
## Purpose
Concept being demonstrated:
Prerequisites and permissions:
Account/region (keep private where needed):
Estimated duration and budget:
Services and expected cost dimensions:
## Plan
Resources to create:
Dependencies and naming/tags:
Steps:
Expected success behavior:
Controlled failure to test:
Abort conditions:
## Results
Observed behavior:
Validation evidence:
Failure diagnosis and repair:
Unexpected costs or limits:
## Cleanup
Resources removed:
Resources intentionally retained, owner, expiry:
Residual-cost verification time:
What I learned:
```

## 3. Architecture decision record

```markdown
# ADR-<number>: <decision>
Status: proposed | accepted | superseded
Date and owner:
## Context
User/business need:
Traffic/data assumptions:
Security/compliance constraints:
Availability and recovery objectives:
Budget/skills/time constraints:
## Options
Option A: benefits, risks, cost, operational effort
Option B: benefits, risks, cost, operational effort
Option C: simpler baseline if applicable
## Decision
Chosen option and why it best meets the requirements:
## Consequences
What improves:
What becomes harder:
Known limitations:
Migration/exit considerations:
## Validation and review
Evidence needed:
Review trigger:
Related diagrams, experiments, and sources:
```

A decision should be falsifiable: define what new requirement or evidence would make you revisit it. Avoid using “best practice” as the complete justification.

## 4. Workload design brief

```markdown
# Workload design: <name>
Users and critical journeys:
Scope and explicit exclusions:
Functional requirements:
Latency/throughput/concurrency targets:
Data volume, growth, retention, and location:
Availability target and measurement window:
RTO/RPO by failure scenario:
Identity and tenant boundaries:
Request/data-flow diagram:
Resource/account/region boundaries:
Data model and access patterns:
Scaling and overload behavior:
Deployment and rollback:
Telemetry, alerts, ownership:
Backup, restore, and failback:
Cost model and unit cost:
Key alternatives and ADRs:
Unresolved assumptions and validation plan:
```

## 5. Threat model

```markdown
# Threat model: <workload>
Assets: data, credentials, compute, availability, financial exposure
Actors: users, operators, services, external attackers, compromised identities
Entry points:
Trust boundaries:
Data classification and retention:
## Threat register
| Threat | Attack path | Impact | Existing control | Evidence | Gap/owner |
|---|---|---|---|---|---|
| Example: unauthorized tenant read | API object ID lookup | Data exposure | Server-side ownership check | Negative test | ... |
## Key questions
How are identities authenticated and authorized?
Where can untrusted input affect queries, paths, commands, or model tools?
Which secrets exist, who can retrieve them, and how are they rotated?
What is encrypted, and who controls/decrypts the keys?
How is abuse detected and contained?
What remains if an administrator or workload role is compromised?
How are controls tested and reviewed after changes?
```

For an AI workload, explicitly include malicious retrieved documents, unsafe tool arguments, cross-user retrieval, model-output handling, and cost amplification.

## 6. Operating runbook

```markdown
# Runbook: <symptom or operation>
Owner and escalation:
Trigger and user impact:
Required permissions/tools:
Account/region/resource identification:
Safety/abort conditions:
## Diagnose
1. Confirm scope and timestamps.
2. Inspect named metrics/logs/traces and recent changes.
3. Test hypotheses in order of likelihood and reversibility.
## Mitigate
Exact action, expected effect, and verification:
Rollback for the mitigation:
## Recover
Restore normal service and validate user journeys:
Check data integrity and backlog:
## Close
Record timeline/evidence:
Remove temporary changes:
Follow-up owner and due date:
```

Do not include commands that operate on ambiguous resources. Identify inputs and expected output, and explain what to do if a check fails.

## 7. Incident review

```markdown
# Incident review: <title>
Date, severity, owner:
Summary and customer impact:
Affected services/accounts/regions:
Detection method and detection delay:
Timeline (one time zone, preferably UTC for correlation):
Trigger:
Contributing technical/organizational conditions:
What evidence supports the explanation:
Mitigation and recovery:
Data integrity/reconciliation outcome:
What worked:
What slowed response:
## Actions
| Action | Risk addressed | Owner | Due | Verification |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
```

Describe system conditions and decisions without blaming individuals. “Be more careful” is not a testable control improvement.

## 8. Cost worksheet

```markdown
# Cost model: <workload>
Pricing review date:
Region/currency/tax assumptions:
Usage scenarios: idle, expected, peak, forgotten-lab month
## Resources
| Resource | Quantity | Billing unit | Usage assumption | Current rate source | Estimated cost |
|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... |
Compute/runtime:
Storage and retained copies:
Requests/throughput:
Network and data transfer:
Observability/security tooling:
Licensing/subscriptions/support allocation:
Credits/discount assumptions (separate from gross cost):
Unit cost: total cost / useful completed units
Top uncertainties:
Budget alert thresholds and owner:
Cleanup and residual-cost checks:
```

Keep gross cost visible even if credits cover the experiment. A temporary credit does not demonstrate long-term affordability.

## 9. Backup and restore test

```markdown
# Restore test: <workload and failure>
Scope and authorized sandbox:
Target RTO/RPO:
Backup source, creation time, retention, encryption:
Recovery target and dependencies:
Known test records/checksums:
Start timestamp:
Restore actions:
Application validation timestamp:
Observed recovery duration:
Newest recovered record and measured data loss:
Identity/key/network issues encountered:
Failback/reconciliation procedure:
Result versus objective:
Retained artifacts and cleanup:
Next test and owner:
```

## 10. Data contract and AI evaluation

```markdown
# Data contract: <dataset/event>
Owner and consumers:
Schema/version:
Required fields and types:
Business key and deduplication rule:
Event-time and timezone semantics:
Freshness/completeness/quality targets:
Allowed schema evolution:
Retention and deletion:
Sensitivity and access controls:
Replay/backfill behavior:
Validation and quarantine rules:
```

```markdown
# AI evaluation: <feature/version>
User task and simpler baseline:
Model/configuration/version:
Dataset provenance and access:
Evaluation cases and held-out split:
Quality rubric and human-review process:
Groundedness and citation criteria:
Unanswerable/adversarial/authorization cases:
Latency and cost measurements:
Failure severity and escalation:
Results by case category:
Regression thresholds:
Known limitations and release decision:
```

## 11. Project README outline

```markdown
# <Project name>
Problem and intended users:
Architecture diagram and key decisions:
Implemented scope and limitations:
Prerequisites, account/region, estimated cost:
Configuration variables (no secrets):
Deployment procedure:
Validation and failure drills:
Operating and recovery instructions:
Measured results:
Cleanup procedure and retained resources:
Future improvements:
Official references and review date:
```

The test of these templates is usability: another authorized engineer should be able to understand your assumptions, repeat the important checks, and identify remaining risks without reconstructing your chat history.
