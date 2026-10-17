# Start here: how to run the tutorials

[Tutorial index](README.md) · [Coverage register](COVERAGE.md) · [Original roadmap](../README.md)

This collection covers the full existing AWS roadmap, including its prerequisite, service, production, specialist, AI, project, career, and reference topics. Related subtopics share coherent tutorials rather than becoming hundreds of disconnected one-page files. Each subject index contains a coverage table.

## 1. Understand the three exercise modes

**Local exercises** use synthetic data and ordinary tools on your computer. They demonstrate programming, data, and failure concepts without provisioning AWS infrastructure. A local process can still call a paid external dependency if the exercise explicitly enables one.

**AWS sandbox exercises** describe actions you may run in an authorized account. Check your principal, account, region, capabilities, permissions, quota, current service availability, and estimated costs first. The authoring/verification of these files has not deployed the cloud infrastructure on your behalf.

**Worked design exercises** produce concrete diagrams, calculations, policy truth tables, migration plans, or recovery strategies. They are used where meaningful live testing requires enterprise accounts, physical hardware, third-party systems, specialist data, or substantial cost. They include validation criteria for a later approved deployment and are not labeled as live tests.

Retired or restricted services appear as lifecycle/recognition/migration topics. A tutorial should not require a new customer to deploy a discontinued product merely to check a box.

## 2. Prepare a dedicated learner workspace

Keep the tutorial Markdown separate from your exercise artifacts. Use a new directory for each lab with code, synthetic fixtures, infrastructure, tests, and evidence. Do not serve a directory containing private files with a demonstration HTTP server. Do not run cleanup commands against a shared project.

Most shell examples use PowerShell; Bash blocks are explicitly labeled. Do not mix quoting, variable, continuation, or path syntax between shells. Python blocks can be complete programs, modules, or clearly described continuations; follow the file-saving instructions before running a fragment.

Use a current supported Python and create a virtual environment when dependencies are needed. Call its interpreter directly on Windows if you do not want to activate a shell script. Record versions and lock dependencies for a reproducible project. Install only tools needed for the current lesson.

## 3. Establish access and costs before cloud work

Complete [account setup](02-account-and-tooling/01-secure-account-and-workstation.md). Use non-root everyday access, temporary credentials where supported, and an explicit account check before writes. Never paste real secrets into the examples or commit credential/state files without reviewing them.

Review the current account plan and service restrictions. Older Free Tier assumptions, historical service names, old model IDs, and outdated tool commands can make a tutorial fail or cost more than expected. Follow the current official links and your account's actual settings.

Set a per-lab budget, time limit, test size, and resource ledger. Ordinary cost alerts are not a universal immediate spending cap. Inspect any account/project-specific spend controls separately. Include idle resources, requests, transfer, storage copies, logs, endpoints, and model inference.

## 4. Follow the practical loop

1. State the problem and expected observable behavior.
2. Read prerequisites and draw the relevant data and permission paths.
3. Prepare a small synthetic fixture and record initial state.
4. Perform the numbered procedure; stop when a prerequisite/API step fails.
5. Verify success using a concrete result, not only a green console status.
6. Run the specified negative/failure case in the dedicated environment.
7. Explain the mechanism, repair it, and preserve sanitized evidence.
8. Clean up exact resources and inspect retained dependencies/costs.
9. Record what remains uncertain and the next useful experiment.

An SDK mock or local simulation cannot prove actual AWS policy evaluation, network reachability, regional support, or all managed-service semantics. Likewise, a one-minute cloud demonstration cannot prove annual availability or production-scale performance.

## 5. Choose your first sequence

For beginners: prerequisite tutorials → account/tooling → cloud foundations → identity → networking → compute/storage/databases/security. Then choose serverless or containers, add IaC and delivery, and continue through operations/reliability/cost.

For AI-focused learners: retain the same identity/network/data/security foundation, then follow the [18-part AI sequence](15-ai-ml/README.md). Start with local ML/retrieval/evaluation before adding managed endpoints, stores, or agent runtimes.

For experienced engineers: use [COVERAGE.md](COVERAGE.md) and the subject indexes to identify gaps, then demonstrate the relevant exercise rather than rereading every introduction.

## 6. Record evidence honestly

Use [engineering documentation practice](23-practice-and-reference/04-documentation-and-maintenance.md) and the existing templates. Mark each stage `local verified`, `AWS sandbox verified`, `design completed`, or `not yet executed` in your own learning log. Never convert a provided example or proposed experiment into a claimed test result.

Check service/model/tool lifecycle before execution and keep the source URL/review date for important changing assumptions. The roadmap and tutorials are a substantial curriculum, not a static replacement for every AWS API reference and regional exception.
