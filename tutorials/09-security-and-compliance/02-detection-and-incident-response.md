# Tutorial 09.2 — Route a sample finding and preserve incident evidence

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../09-security-and-compliance.md)

**Mode:** optional GuardDuty sample-finding lab plus an incident tabletop. No malicious traffic or attack deployment. **Result:** triage evidence, a scoped event route and a response playbook. Reviewed 2026-10-18.

## 1. Decide what question each service answers

Use one fictional event: an application role is used unexpectedly and a sensitive file may have been read. CloudTrail can identify recorded API activity; GuardDuty can flag supported suspicious behavior; Config can show relevant configuration; Macie can help classify supported S3 data; Detective can supply investigation context.

Inspector addresses covered vulnerabilities/exposures, not every malicious action. Security Hub/CSPM helps organize posture and findings across enabled scope. Security Lake centralizes supported security data for analysis; central storage alone is not a complete detection program.

Create a table of question, service/data source, required enablement, Region/account coverage, retention and investigator. Do not check every service merely to fill a console dashboard; each adds permissions, cost and operating work.

## 2. Inspect management activity first

In CloudTrail Event history for the sandbox Region, find a recent benign management action you performed, such as creating a dedicated lab role or resource. Record principal/session, event name, timestamp, resource where available and request ID. Do not copy tokens or full sensitive parameters into public notes.

Explain the limits: management event history is not a guarantee that an S3 object read was logged. Data-event selectors must cover relevant resource operations when that evidence is required. Event history is not your long-term centralized archive.

For a trail design, choose Regions, management/data events, S3 destination, access controls, encryption, integrity validation, lifecycle and optional integrations. Test delivery and retention. Duplicated/high-volume data collection can cost more than expected, so scope it to the threat/evidence requirement.

## 3. Use only an authorized GuardDuty scope

If the sandbox is centrally managed, use its existing setup and do not alter organization controls. Otherwise, review the selected Region's GuardDuty features and pricing before enabling a dedicated learning deployment. Note which optional protections are enabled; foundational detection does not imply every workload feature is covered.

Use the console's sample-findings function. Samples have placeholder data and a sample marker. Select one finding, inspect its JSON and record type, severity, account, Region, resource description and time. Write `sample, not a confirmed incident` at the top of your report.

A sample tests data handling and triage. It does not demonstrate that a real malicious action would be detected by every protection plan. No tutorial step requires launching a vulnerable instance or attacking an endpoint.

## 4. Route the sample to an inspectable destination

Optionally create a dedicated EventBridge rule matching GuardDuty findings in the intended account/Region. Choose a dedicated CloudWatch Logs target and configure the required resource permissions using the supported console workflow. Keep all remediation targets disabled.

Generate the sample again as supported, then inspect the destination for the event. Correlate finding ID and timestamps. If routing does not occur immediately, inspect event-pattern matching, target permission, delivery failures and the service's documented event behavior before generating many more samples.

Negative test the pattern locally or with the console's event-pattern testing tool using a fabricated unrelated event. It should not match. A broad `source` match without resource/account constraints may be too wide for a future automated containment action.

## 5. Triage using evidence and hypotheses

Write four sections: observed fields, hypotheses, missing evidence, proposed next action. For a fictional credential-use finding, seek session identity/source, associated API events, affected resources, source-network context and owner confirmation. A severity score helps prioritize; it does not establish the business impact by itself.

Compare a vulnerability finding: investigate affected package/version, exploitability in context, exposure, fix and compensating controls. Compare sensitive-data discovery: verify classification scope and access exposure. These require different owners and evidence even when displayed in one findings tool.

For Config/CSPM, confirm resource recording and account/Region scope. A missing finding may mean missing coverage, not a compliant resource. Document inherited controls and disabled/excluded checks instead of treating an aggregate score as proof of compliance.

## 6. Run an evidence-preserving tabletop

Scenario: the application role may be compromised and an instance may be running unauthorized code. Write a timeline beginning with alert receipt and identify the incident commander, platform investigator and business owner.

Contain identity access using supported session/permission controls, considering the effect on valid workloads. Isolate the affected instance using a planned method that preserves necessary forensic access. Avoid terminating it immediately if doing so destroys evidence. Record snapshot/log preservation, access controls, timestamps and chain of custody.

List what would confirm scope: CloudTrail activity, application request IDs, storage access events if configured, network metadata, process/host evidence and deployment provenance. VPC Flow Logs are not packet payloads; log gaps must be stated rather than filled with assumptions.

Then eradicate and recover: rotate affected secrets, rebuild from trusted artifacts, restore verified data if needed, reconnect safely and monitor for recurrence. Merely restarting a suspicious process is not a demonstrated removal of the cause.

## 7. Add five incident variants

For exposed data, restrict access while preserving evidence and assess the actual access scope. For a malicious dependency, identify deployed artifact versions and rebuild from a trusted dependency set. For ransomware-like deletion, protect independent recovery copies and validate keys before restore. For denial of service, inspect traffic, application bottlenecks and protection controls. For credential leakage, revoke the credential/session path and find dependent workloads needing recovery.

For each variant, specify a detection trigger, one containment action, one destructive-action risk and a verification gate. Keep communications as a documented plan; this exercise does not send notifications to real customers or colleagues.

## 8. Bound automation and review

Start automation with enrichment and scoped routing. Before resource remediation, define idempotency, retries, dead-letter handling, authorization, account/resource allowlists, approval policy where your organization requires it, rollback and audit evidence. A sample finding must never terminate unrelated instances.

Measure detection, triage, containment and recovery times separately. The post-incident review should produce a specific control/runbook improvement with owner and due date, not blame or a generic “be more careful.”

## Verify and clean up

Pass with sample identification, scoped event delivery, an unrelated-event negative check and a playbook that preserves evidence. For missing events, distinguish source coverage, event matching, permissions, Region and latency. For noisy findings, refine scope/suppression with evidence rather than disabling all detection.

**Cleanup/cost:** archive only sample findings, remove dedicated event rules/targets/logs and disable only services enabled solely for your disposable lab if policy permits. Preserve pre-existing security baselines. CloudTrail Lake/Audit Manager maintenance means new learning should use supported trails and current evidence workflows. Analysis, logs, exports and integrations may charge independently.

Sources: [sample findings](https://docs.aws.amazon.com/guardduty/latest/ug/sample_findings.html), [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html), [Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html), [incident response guide](https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/), [maintenance register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
