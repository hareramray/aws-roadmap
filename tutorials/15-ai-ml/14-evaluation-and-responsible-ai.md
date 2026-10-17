# 14 — Build an evaluation harness and a release decision

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** local evaluation first; optional AWS/model runs using the same fixtures. **Result:** reproducible checks, a human rubric, segment-level results, and an evidence-based release decision. This tutorial complements statistical metrics in [ML foundations](01-ml-foundations.md).

## 1. Define the unit of success

For the support assistant, success means answering a permitted question from current evidence, citing the supporting passage, and abstaining when evidence is missing. Security failures have higher severity than a minor wording issue. Define latency and cost limits separately so a high-quality answer does not hide an unusable system.

Create a fixture format with case ID, authenticated test tenant, input, expected evidence IDs, expected abstention, category, and severity. Keep a development set for tuning and a held-out set for release checks. Do not repeatedly adjust prompts based on the held-out answers and continue calling it unseen evaluation.

## 2. Run deterministic contract checks

Save as `evaluation_lab.py`. This local code evaluates canned outputs; replace them with your application's captured structured responses after the contract is stable.

```python
CASES = [
    {"id": "refund", "evidence": {"blue-refunds:v2"}, "abstain": False},
    {"id": "unknown", "evidence": set(), "abstain": True},
    {"id": "unauthorized", "evidence": set(), "abstain": True},
]
OUTPUTS = {
    "refund": {"answer": "Thirty days.", "citations": ["blue-refunds:v2"], "abstain": False},
    "unknown": {"answer": "Insufficient evidence.", "citations": [], "abstain": True},
    "unauthorized": {"answer": "Cannot provide that record.", "citations": [], "abstain": True},
}


def check(case, output):
    required = {"answer", "citations", "abstain"}
    schema_ok = (isinstance(output, dict) and set(output) == required
                 and isinstance(output["answer"], str)
                 and isinstance(output["citations"], list)
                 and all(isinstance(x, str) for x in output["citations"])
                 and isinstance(output["abstain"], bool))
    if not schema_ok:
        return False
    citations = set(output["citations"])
    return (output["abstain"] == case["abstain"]
            and citations <= case["evidence"]
            and (case["abstain"] or bool(citations)))


def main():
    results = {case["id"]: check(case, OUTPUTS[case["id"]]) for case in CASES}
    assert all(results.values())
    bad = {"answer": "Seven days.", "citations": ["red-refunds:v1"], "abstain": False}
    assert not check(CASES[0], bad)
    print(results)
    print("PASS: contract checks and wrong-evidence negative case")


if __name__ == "__main__":
    main()
```

Expected: all three fixture checks pass and the wrong-evidence output fails. These checks verify format, citation membership, and expected abstention. They do **not** prove “Thirty days” is supported by the cited text. Add a human or separately validated groundedness evaluation for that claim.

## 3. Measure retrieval separately

For each answerable query, label the document/chunk IDs that contain the needed evidence. Compute recall at `k`: how much required evidence appears in the retrieved set? Also inspect precision: how much returned material is relevant? The exact unit (document or chunk) must stay consistent across experiments.

Use the same fixture set to compare chunk sizes, lexical/vector/hybrid search, retrieval depth, and reranking. Change one factor at a time. Log the source snapshot and index/model versions. A retrieval improvement can reduce generation errors without changing the model.

## 4. Add an explicit human rubric

Score each response on correctness, support from evidence, completeness, relevance, clarity, appropriate abstention, and harmful or unauthorized exposure. Define concrete anchors: `0 = unsupported/wrong`, `1 = partly supported with material omission`, `2 = fully supported for the task`. Keep security pass/fail separate so averages cannot hide a data leak.

Have two reviewers independently score a small overlap. Discuss disagreements and refine the rubric before scaling. Redact identifying information where possible without destroying the evaluation context. Record when reviewers need domain expertise and when an answer must be escalated to a qualified person.

For model-based judging, test agreement with human labels and include adversarial outputs that try to influence the judge. Keep judge instructions/evidence separate from the candidate's untrusted text. Judge models can exhibit bias and correlated errors; a numerical score is not automatically an objective truth.

## 5. Include responsible-AI scenarios

Build cases for representative languages, accessibility needs, ambiguous wording, incomplete input, and relevant population segments. For predictive models, compare error rates/calibration across appropriate segments with enough data and a justified purpose. Avoid inferring sensitive attributes merely to populate a report.

Document data provenance, consent/licensing, intended uses, prohibited uses, known limitations, human review, correction/appeal paths, and retention. Define who owns a harmful output incident and how affected data/models/configuration would be investigated. A responsible-AI statement needs tested controls and operating ownership.

For synthetic test examples, label them as synthetic and avoid presenting them as evidence of real-world fairness. Small lab samples teach the process; they cannot establish broad deployment safety. [AWS responsible AI](https://aws.amazon.com/ai/responsible-ai/)

## 6. Compare two candidates fairly

Freeze the prompt/data/tool configuration for candidate A and B. Run the same case set, account for output randomness, and measure task success, severity-weighted failures, latency distribution, token usage, and cost. Repeat stochastic cases enough to detect unstable behavior; record the number of trials.

A release gate might require zero observed authorization violations, a minimum grounded-answer rate on your fixture set, and no significant regression in p95 latency or cost per successful task. These are project-specific thresholds, not universal AWS standards. Explain the uncertainty left by the size and realism of the test set.

Optional managed evaluation should use supported Bedrock/AgentCore evaluation capabilities for the chosen workflow, with restricted dataset access and estimated job/model costs. Keep the raw fixtures and rubric portable so a service change does not erase your evidence. [Bedrock evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)

## 7. Release, monitor, and recover

Version the approved model identifier, prompt, tool schema, guardrail version, index snapshot/configuration, and application artifact. Roll out to a bounded audience and observe real task failures with privacy-aware sampling. Define who can stop rollout and what version is restored.

Keep user feedback separate from verified labels; a thumbs-up is not proof of factual correctness. Review failed/abstained cases and add representative new cases to future development sets, preserving a clean release set.

## Failure exercise, cost, and cleanup

Deliberately change the canned Blue answer to an unsupported statement while leaving its citation unchanged. Observe that contract checks may still pass. Add a claim-support review that catches it. This demonstrates why a superficial automated pass is incomplete.

Delete temporary model-evaluation jobs/storage where appropriate, retain only sanitized cases/results needed for reproducibility, and set retention on traces. Model judging multiplies inference cost; estimate judge calls, retries, evaluation data storage, and human review effort.

**Completion:** deliver the fixture schema, baseline/candidate report, rubric, negative-test results, remaining uncertainties, and a justified release or no-release decision.
