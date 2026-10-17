# Tutorial 09.3 — Turn a threat model into tested controls and evidence

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../09-security-and-compliance.md)

**Mode:** worked security design and tabletop; optional count-mode WAF inspection on a dedicated lab endpoint. **Result:** threat register, control tests and compliance-evidence map. Reviewed 2026-10-18.

## 1. Draw assets and trust boundaries

Use the document portal: browser, authentication provider, public API/edge, application runtime, metadata database, private object storage, CI system and backup destination. Label boundaries between customer, application, administrator, AWS account and external provider.

List assets beyond files: deployment authority, secrets, encryption keys, domain registration, logs, backups and service availability. A stolen deployment credential can alter code that later reads data, even if the data bucket itself remains private.

Create a threat register with scenario, entry point, affected asset, likelihood/impact rationale, prevention, detection, response, owner and review date. Use concrete examples: Alice requests Bob's object; a leaked CI token deploys modified code; repeated large uploads exhaust processing capacity; a privileged role deletes recovery data.

## 2. Test layered controls

For cross-user access, require application object ownership checks and restricted storage delegation. Negative test: Alice's authenticated request for `doc-b` returns a denial without revealing its content. TLS and encryption at rest do not fix a missing ownership check.

For malicious deployment, restrict CI OIDC trust, protect release branches/environments, use separate deployment/runtime roles and limit `PassRole`. Verify that an unapproved repository/branch cannot assume the deployment role. Inspect artifact provenance, dependency/image scans and release version, recognizing that scans cover particular checks rather than every business flaw.

For upload exhaustion, constrain size/type, authenticate admission, bound concurrency and observe backlog/age. A WAF rate rule can contribute but cannot know every expensive application operation's business cost. Design application-level limits and response behavior too.

## 3. Design account and organization boundaries

For an established company, separate production, nonproduction, log archive and security administration where justified. Define an OU placement and delegated security administrator. Keep ordinary workloads out of the management account.

Use a **local** guardrail exercise: an SCP restricts certain administrative actions in development; an RCP restricts supported resource access; runtime identity policies still grant the intended application actions. Test whether emergency response and legitimate service integrations remain possible. Do not deploy an unreviewed broad deny across an organization.

Root recovery uses protected channels and MFA; break-glass access has a named owner and tested procedure. Workforce federation and temporary roles are the everyday path. For a personal sandbox, retain a simpler setup and do not create an organization without reviewing account-plan consequences.

## 4. Build a data-perimeter matrix

Use rows for application role, CI role, security investigator and AWS replication/service role. Use columns for source objects, database, logs and backups. Mark required read/write/admin operations and expected network/source context.

Then test a request through the private endpoint and an otherwise identical request through another path. Endpoint policy alone does not grant access or close other permitted paths; coordinate identity/resource/organization policies where needed. Service-to-service calls may carry different context, so broad source-IP/VPC conditions need explicit compatibility tests.

Add IPv6 to the network matrix. Security-group and NACL rules, routes, DNS and inspection must cover the actual address family. A perfectly restricted IPv4 path does not secure an accidentally open IPv6 listener.

## 5. Work through WAF configuration safely

For a dedicated lab HTTP application, define an optional current AWS WAF web ACL with a managed-rule group in count/evaluation mode and a deliberately scoped custom rule for a harmless test header such as `X-Training-Case: observe`. Choose the correct Regional/global scope for the protected resource.

Send an ordinary request to your own endpoint with and without the header. Inspect sampled requests/logs and rule labels. Expected: count mode records the match while the application still receives the request. A count observation is not evidence that blocking is enabled.

Define a rate-based rule using realistic expected client behavior and current supported aggregation/window settings. Test the logic with a design/sample rather than generating abusive traffic. Before production blocking, review false positives, exclusions, rule order and the effect on authenticated users behind shared addresses.

Use current WAF APIs, not WAF Classic. WAF inspects supported application requests; it is not a replacement for database authorization, secure code or every transport-level protection.

## 6. Distinguish perimeter services

For DDoS resilience, identify how the chosen Shield tier and service integrations address the workload, and what the incident engagement/cost terms require. Do not infer that every protection tier has identical features.

For Network Firewall, define inspected routes, stateful policy and symmetric return path. For Firewall Manager, define organizational policy scope and delegated operations. For Resolver DNS Firewall, define allowed/blocked DNS behavior through the supported resolver path. These are different controls: DNS filtering is not general packet-content inspection, and centralized policy administration is not itself an application firewall.

Add security groups for workload communication and NACLs only where their subnet-level purpose is clear. A group reference is not transitive application authorization. Keep internal database TLS and least privilege even behind private routing.

## 7. Convert requirements into evidence

Use a fictional policy requiring private customer data, controlled changes, 30-day operational log retention and quarterly recovery tests. Build an evidence table:

| Requirement | Control | Evidence | Owner |
| --- | --- | --- | --- |
| Customer isolation | Per-object authorization and scoped storage access | Alice/Bob negative test and policy review | Application owner |
| Controlled release | Reviewed pipeline and restricted federation | Artifact/version, approval record, deployment event | Delivery owner |
| Log retention | Protected destination and lifecycle | Configuration plus sample retrieval/deletion timing | Platform owner |
| Recoverability | Independent backups and restore exercise | Checksums, application queries, measured RTO/RPO | Service owner |

Use Artifact for relevant AWS reports/agreements and identify inherited controls. Customer configuration, procedures and application behavior need their own evidence. A service being eligible for a standard does not certify the complete workload.

Document data classification, location, retention/deletion, access reviews, vulnerability remediation and incident ownership. Resolve contradictory retention/deletion requirements with the responsible organization rather than inventing legal conclusions in a technical template.

## 8. Review scope and exceptions

Check every account, Region, resource type and excluded control. A clean dashboard can simply omit the unrecorded resource. Record temporary exceptions with rationale, compensating control, owner and expiry; do not suppress a finding indefinitely because it is inconvenient.

Run a tabletop where logs stop arriving while the dashboard still shows yesterday's status. The control must detect delivery interruption or stale evidence, not just report the last successful check. Add a second tabletop where emergency containment conflicts with a guardrail; document the intended authorized response path.

## Verify and clean up

Pass with four concrete threats, positive/negative tests, a data-perimeter matrix and evidence owners. If a control cannot produce evidence, refine its objective. If a WAF rule blocks legitimate traffic, inspect exact matching/rule order and roll back the dedicated change rather than disabling unrelated protections.

**Cleanup/cost:** design-only work creates no resources. Remove any dedicated web ACL association/rules/log group and test endpoint created for the optional exercise, preserving shared baselines. WAF rules/requests/logging and advanced protection tiers have separate costs. No tutorial requires changing production organization controls.

Sources: [WAF guide](https://docs.aws.amazon.com/waf/latest/developerguide/), [Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/), [Artifact](https://docs.aws.amazon.com/artifact/latest/ug/), [shared responsibility](https://aws.amazon.com/compliance/shared-responsibility-model/).
