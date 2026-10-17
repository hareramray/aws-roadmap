# 03 — Turn the curriculum into a study plan and demonstrable skills

[Practice index](README.md) · [All tutorials](../README.md)

**Mode:** local planning, explanation practice, and project review. **Result:** a role-based schedule, objective-to-evidence tracker, current exam-scope check, and a technical interview portfolio. This implements the learning/career topics in root chapters 00, 24, 25, 26, and 27.

## 1. Assess your starting point using demonstrations

Pick six tasks: explain an HTTP request path, run a local app, use Git, write a transaction, identify an AWS principal, and explain an idempotent retry. Score each `0 unknown`, `1 explain`, `2 build`, or `3 operate/debug`. Attach evidence rather than rating confidence alone.

Use the [prerequisite tutorials](../01-prerequisites/README.md) for weak areas. An experienced developer may move quickly through code basics but still need cloud identity/networking. An administrator may be comfortable with Linux but need data modeling and delivery pipelines.

## 2. Create a weekly learning loop

Choose a realistic time allowance. Use the root [24-week example](../../00-learning-plan.md) as a starting schedule, then assign actual tutorial files to weeks. Reserve explicit time for reading, implementation, deliberate failure, explanation, and cleanup.

For week one, a concrete sequence is: run the local HTTP diagnostic exercise; build the SQLite transaction; complete the identity/account check if a sandbox is ready; write a short journal entry explaining one misconception. Do not replace the failure/cleanup session with another video just to increase topic count.

At the end of each week, record what you can now demonstrate, unresolved questions, retained resources, and the next smallest useful experiment. Revisit failed checks before adding unrelated services.

## 3. Select a role and deepen the relevant path

Map your target role to workload evidence. CloudOps emphasizes diagnosis, patching, access, recovery, and costs. Development emphasizes APIs, SDKs, events, data integrity, testing, and delivery. Architecture emphasizes requirements, tradeoffs, reliability, governance, migration, and economics. Security emphasizes trust boundaries, prevention/detection, evidence, and response. Data/AI paths require data contracts, evaluation, deployment, and operations as well as the shared foundation.

Choose two capstones aligned with the role, plus one exercise that exposes a weak foundational area. Keep a broad survey of specialist domains while accurately labeling which skills you have only studied.

## 4. Verify certification scope before buying preparation

Open the current [AWS exam-guide index](https://docs.aws.amazon.com/aws-certification/latest/examguides/). Choose the exact current exam code, language, availability, and target candidate scope. Check updates/retirement notices on the certification page. Use the root [certification chapter](../../24-certifications-and-career-paths.md) as a dated guide, not a replacement for current booking information.

Create a table with official objective, roadmap/tutorial link, evidence, practice-question mistake, and next action. Copy current domain weights into your private tracker from the official guide; do not assume an old course matches the present version.

For each practice question, explain why the chosen answer meets the stated constraints and why alternatives fail. Categorize mistakes as missing service knowledge, missed requirement, confused mechanism, or time pressure. Use legitimate official/sample preparation rather than recalled exam dumps.

## 5. Practice explanations with a concrete system

Pick one completed project and answer: who uses it, what happens on a request, where state lives, which identity can access it, how it scales, what fails, how you know, how you recover, and what costs money. Draw the data path, permission path, and recovery path separately.

Use the root [question bank](../../25-interview-and-design-questions.md). For every answer, state assumptions, mechanism, tradeoff, and verification. Replace “use Multi-AZ” with an explanation of the particular deployment type, failure scenario, application reconnection behavior, and remaining regional risks.

Prepare a five-minute walkthrough and a deeper technical version. Distinguish what you personally built from generated scaffolding and borrowed reference patterns. A learning lab should be described as a lab, not as commercial production experience.

## 6. Run a system-design interview exercise

Choose appointment booking, image processing, a multi-region API, enterprise analytics, or a knowledge assistant from the root questions. Spend an initial period clarifying users, data, load, latency, availability, recovery, security, and budget. Write assumptions explicitly.

Estimate rough capacity, define APIs and data access patterns, draw the simplest viable design, then introduce one failure and one scale change. Explain the chosen consistency and authorization behavior. Compare an alternative and record what requirement would make you switch.

End with testability: what measurement, prototype, restore drill, or current service documentation would resolve your largest uncertainty? An honest uncertainty plus a concrete validation plan is stronger than an invented guarantee.

## 7. Use the glossary actively

Take ten terms from [the glossary](../../27-glossary-and-mental-models.md). For each, write a definition, a neighboring term it is often confused with, and an example from your project. Examples: availability versus durability, authentication versus authorization, replica versus backup, message ID versus receipt handle, SLO versus SLA, drift versus data leakage.

Explain them aloud without reading. If you can name the service but cannot describe its failure or security boundary, revisit the worked tutorial. Use the [master checklist](../../26-master-completion-checklist.md) as evidence gates.

## Completion

Deliver a realistic schedule, role/evidence matrix, current exam-scope record if relevant, two project summaries, one design exercise, and a list of remaining gaps. Do not claim mastery of every specialty merely because every file has been opened. This workflow has no required AWS costs; project experiments retain their own budgets and cleanup obligations.
