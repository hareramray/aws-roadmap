# 04 — Write usable engineering evidence and keep it current

[Practice index](README.md) · [All tutorials](../README.md) · [Templates](../../32-engineering-templates.md)

**Mode:** apply templates to one completed tutorial/project. **Result:** a lab record, architecture decision, threat model, runbook, incident review, cost worksheet, restore report, data/AI contract where relevant, and a source-update record. These are working documents, not decorative paperwork.

## 1. Write a reproducible lab entry

Choose the private S3 exercise or another completed small tutorial. Copy the lab template from root chapter 32. Fill purpose, prerequisites, exact environment, expected behavior, steps, actual results, failure, cleanup, and retained resources. Keep account IDs/private details out of a public copy.

Include the command/configuration that materially explains the outcome, not every terminal line. Record whether a failure was access denied, not found, or network-related. A screenshot of an empty console without account/region context is weak cleanup evidence.

## 2. Turn a choice into an ADR

Compare two plausible designs for your workload. For a small support API, compare synchronous processing with a queue-backed worker. Write requirements and consequences: response semantics, latency, failure/retry behavior, operational overhead, permissions, and cost.

State the decision and a review trigger, such as a change in processing duration or throughput. “AWS recommends it” is not a complete reason; explain how the design fits your requirements. Link a measurement or official capability statement supporting the important constraint.

## 3. Complete a threat model and runbook

List assets, entry points, principals, trust boundaries, threats, controls, and test evidence. For every mitigation, identify an owning layer. Encryption and IAM are different controls; a model guardrail and tool authorization are different controls.

Then write a runbook for one observed failure. Include trigger, owner, required permissions, account/resource identification, diagnostic steps, narrow mitigation, rollback, user recovery checks, and evidence collection. Have another learner follow it or rehearse it later without relying on memory.

## 4. Produce cost and recovery evidence

Populate the cost worksheet with idle, expected, and bounded peak scenarios. Include billing unit, quantity, region, current rate source, assumptions, retained copies, network, logging, and credits separately. Compute one useful unit cost. Identify the largest uncertainty and how to measure it.

Run a restore test from the reliability tutorials. Record known records/checksums, backup time, start time, application validation time, newest recovered data, dependency failures, and observed RTO/RPO. Do not fill the “result” field with the target value before measuring.

Write an incident review from the troubleshooting workshop: impact, timeline, trigger, contributing conditions, mitigation, integrity outcome, and assigned corrective actions with verification. Avoid blame and untestable actions such as “be more careful.”

## 5. Add data and AI documentation where relevant

For a pipeline, document schema, business keys, event-time meaning, duplicates, quality, retention, access, evolution, and replay/backfill. For an AI application, document task/baseline, model/configuration, dataset provenance, fixtures, rubric, security cases, quality/latency/cost, limitations, and release criteria.

Connect every claimed result to a saved artifact. A model version alone is insufficient if prompts, retrieval data, chunking, tools, or guardrails changed. A data pipeline's code commit alone is insufficient if the source snapshot and schema are unknown.

## 6. Check sources and lifecycle

Use [the official resource map](../../28-official-resources-and-updates.md) and [service coverage map](../../30-service-coverage-map.md). Open the primary guide for the exact feature, then inspect API/security/quota/region/pricing/lifecycle details. A visible documentation page or SDK method does not prove new-customer availability.

Record claim, source URL, review date, applicable account/region/version, and review trigger. Distinguish launch announcements from maintained operational documentation. If sources conflict, preserve the exact scoped statements and resolve the operational ambiguity before building; do not silently choose the convenient one.

Review services in [maintenance](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html), [sunset](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html), and [shutdown](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html). Historical products belong in recognition/migration exercises, not fresh deployment instructions.

## 7. Make the project readable

Write the README as if the reviewer has never seen your conversation: problem, implemented scope, architecture, prerequisites/cost, configuration without secrets, setup, validation, operations/recovery, measured results, cleanup, limitations, and sources. Link the detailed evidence rather than repeating everything.

Check relative links, balanced code fences, valid JSON/YAML examples, and language syntax. Label illustrative snippets versus complete programs and actual deployments versus design exercises. Do not replace unknown values with invented account IDs, model identifiers, prices, or benchmark results.

## 8. Maintain the collection

Periodically recheck the tools/services your projects actually use: runtimes, SDKs, database/Kubernetes versions, service lifecycle, prices, quotas, models, and exam scope. Update the affected instructions and rerun relevant checks. Keep a concise change record of the old assumption, new evidence, changed files, and validation.

**Completion:** another authorized reader can understand the project, reproduce its important evidence, identify what remains untested, and clean up its dedicated resources without reconstructing your chat history.
