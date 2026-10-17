# 02 — Practice fifteen diagnostic scenarios

[Practice index](README.md) · [All tutorials](../README.md) · [Detailed diagnostic reference](../../31-troubleshooting-playbook.md)

Use disposable local or sandbox systems from the subject tutorials. Introduce only one reversible fault at a time, record the original configuration, set an experiment deadline, and preserve enough evidence to explain the cause. Never inject these faults into an unrelated production system.

For every scenario, write: expected behavior, actual symptom, scope/time, first discriminating check, evidence, cause, narrow repair, verification, and follow-up. Do not mark success merely because the error disappeared after several unrelated changes.

## 1. Expired identity versus denied permission

Use a test profile/session and compare a session-authentication failure with an intentionally ungranted read. Run `sts get-caller-identity` first. Record whether identity is established, then inspect action/resource/policy context. Refresh the expired session through its supported flow; repair the narrowly missing grant only in the dedicated role. Prove that the repaired read works while unrelated reads remain denied.

## 2. DNS versus route versus listener

Use the local HTTP lesson to compare a nonexistent hostname, closed port, and valid HTTP 404. In an AWS network lab, inspect the client's actual DNS answer and route/security path. Repair the failing layer and repeat the same test from the same client location. A successful DNS lookup does not establish transport reachability.

## 3. TLS mismatch

Review a synthetic certificate/hostname mismatch or an isolated lab listener. Compare requested hostname, certificate names, validity, trust chain, and termination point. Explain why turning off verification hides the problem. Correct the intended certificate/domain binding and verify HTTPS with normal certificate validation.

## 4. Load-balancer health failure

Change the health path in a dedicated service to a nonexistent endpoint. Observe target-health reason and user response while the application process still runs. Restore the correct path and verify business behavior, not only probe status. Extend by comparing no healthy targets with a target-generated 500 error.

## 5. Compute bootstrap failure

In a disposable launch configuration, use a deliberately invalid application command. Observe that instance running state does not imply successful startup. Inspect user-data/bootstrap and process logs. Fix the versioned launch definition, replace the disposable instance, and prove the replacement works without manual hidden repair.

## 6. Private object denied or apparently stale

Compare authenticated/anonymous access to the exact known S3 key. Then change a cached site's object and distinguish origin state from CDN/browser caching. Inspect key case, versions/delete markers, policy, key permissions, and cache headers as relevant. Do not disable public-access blocking to fix a private role's missing permission.

## 7. Database connection or query failure

Compare a wrong credential, a blocked network path, and a deliberately slow query on synthetic data. Observe connection timing/errors, application logs, database sessions/locks/query plan. Repair one cause, rerun the same transaction, and verify data integrity. Avoid increasing instance size without bottleneck evidence.

## 8. Conditional-write conflict or hot key

Submit two conflicting updates to the same synthetic versioned record. Confirm the expected conditional failure protects the invariant. Separately model a skewed key distribution and compare it with evenly distributed traffic. A retry that removes the condition may corrupt correctness even if the error disappears.

## 9. Lambda timeout or throttling

Add a bounded local/sandbox delay to a dependency. Observe duration/concurrency/error changes and the invocation mode's retry behavior. Set a suitable deadline/concurrency policy and fix the slow dependency rather than merely extending timeouts. Verify duplicates do not multiply business results.

## 10. Queue backlog and poison message

Use a worker that rejects one known synthetic payload. Watch receive count, age, retries, and DLQ behavior. Repair the parser/business rule, replay only the intended test messages, and verify idempotency. Do not purge the queue to conceal an unexplained backlog.

## 11. Container image, scheduling, or health

Compare an invalid image reference, insufficient requested capacity, and a container that exits after startup. Inspect service events, task/Pod state, exit code, resource limits, and probes. Fix the corresponding layer and redeploy the known artifact. Preserve the failing logs before replacement.

## 12. IaC or pipeline failure

Create a harmless validation failure in a copied template or a pipeline unit-test failure. Verify it blocks the change before deployment. For a sandbox resource failure, inspect the first meaningful event and retained state; do not delete backend state or force-unlock active work blindly. Correct, review the new plan, and verify resulting state.

## 13. Pipeline data corruption

Add a duplicate business key, malformed row, and late record to a synthetic dataset one at a time. Compare row counts and business aggregates with expected values. Verify quarantine, deterministic reprocessing, and lineage. A row-count match alone may hide wrong values.

## 14. AI retrieval, authorization, or quality failure

Use the [AI evaluation/security fixtures](../15-ai-ml/README.md). Compare a missing relevant chunk, an incorrect cited answer, and an unauthorized document request. Determine whether the failure belongs to retrieval, generation, or access enforcement. A prompt change cannot fix absent evidence or missing authorization.

## 15. Unexpected cost

Use a synthetic cost table or your own small lab's billing view. Group by account/region/service/usage type/time and correlate with resource creation and traffic. Separate usage growth, rate changes, retention, and commitment allocation. Stop/remove only confirmed dedicated causes and inspect delayed billing before claiming the issue is resolved.

## Final incident exercise

Choose one scenario and ask another learner to diagnose it from your runbook without knowing the injected fault. Provide only the symptom and authorized scope. Record detection time, misleading signals, missing permissions/data, recovery time, and integrity checks. If working alone, write the expected diagnostic sequence before injecting the fault and compare it with your actual actions.

Produce a short incident review with timeline, impact, trigger, contributing conditions, mitigation, recovery, and specific follow-up actions. Use UTC for cross-service timestamps or state one chosen time zone consistently. Exclude secrets and unnecessary personal data from escalation evidence.

Restore the original lab configuration or tear down the dedicated resources. Evidence should explain the mechanism and practical limit of the test. A simulated outage is valuable, but it is not proof that every real regional failure has been handled.
