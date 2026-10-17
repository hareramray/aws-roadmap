# 17 — Assemble and evaluate a support assistant

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** complete local baseline, then optional staged AWS implementation. **Result:** a reproducible support assistant with retrieval, authorized order lookup, evaluation, release criteria, and cleanup. This implements [portfolio project 7](../../23-projects-and-portfolio.md), while the predictive-model track uses tutorials 01–05.

## 1. Write the acceptance contract

The assistant answers current refund/shipping questions for a verified tenant and looks up that tenant's synthetic orders. It must abstain when evidence is absent, preserve citations, reject cross-tenant access, limit tool work, and expose enough telemetry for diagnosis. It must not execute purchases, refunds, emails, or arbitrary code.

Create a project with `src`, `tests`, `fixtures`, `infra`, and `docs` directories. Store no secrets. Record the intended profile/region privately, a spend allowance, a request bound, and the resource ledger. Define measurable educational targets based on your fixtures rather than claiming production-scale availability.

## 2. Assemble the local baseline

Save the complete scripts from tutorials 09 and 12 as `retrieval_lab.py` and `tool_boundary.py` in the same working directory. Save this third file as `assistant_lab.py`:

```python
from retrieval_lab import DOCS, build_index, retrieve
from tool_boundary import Principal, lookup_order

INDEX = build_index(DOCS)


def answer_policy(question, principal):
    hits = retrieve(question, principal.tenant, INDEX)
    if not hits:
        return {"answer": "Insufficient authorized evidence.", "citations": [], "abstain": True}
    # Extractive baseline: preserve the exact evidence rather than inventing prose.
    evidence = [chunk for _, chunk in hits]
    return {"answer": " ".join(chunk["text"] for chunk in evidence),
            "citations": [chunk["chunk_id"] for chunk in evidence], "abstain": False}


def main():
    blue = Principal("user-1", "blue", frozenset({"orders:read"}))
    response = answer_policy("refund window", blue)
    assert not response["abstain"]
    assert "thirty days" in response["answer"]
    assert all(citation.startswith("blue-") for citation in response["citations"])
    assert answer_policy("lunar vehicle warranty", blue)["abstain"]
    assert lookup_order({"order_id": "blue-001"}, blue)["status"] == "dispatched"
    try:
        lookup_order({"order_id": "red-002"}, blue)
    except PermissionError:
        print("PASS: policy retrieval, abstention, and authorized order lookup")
    else:
        raise AssertionError("Cross-tenant access unexpectedly succeeded")
    print(response)


if __name__ == "__main__":
    main()
```

Run all three files. Expect their assertion checks to pass. The baseline intentionally uses extractive text and deterministic dispatch; it is not a foundation-model agent. It establishes correct data/access behavior before introducing probabilistic generation.

## 3. Freeze fixtures and measure the baseline

Create at least twelve cases spanning answerable, unanswerable, conflicting-version, malformed-input, unauthorized-order, and prompt-injection categories. Use the schema/rubric from tutorial 14. Preserve expected evidence IDs and access context. Measure exact business outcomes separately from wording.

Add tests for document removal and access revocation. A model-independent baseline should fail the same authorization tests if the underlying control is broken. This makes regression investigation easier than treating every failure as a prompt issue.

## 4. Add one AWS component at a time

First replace local generation with one approved Bedrock model using tutorial 06. Keep local retrieval and tool authorization unchanged. Compare its answers with the extractive baseline, logging model/prompt version and usage. Enforce output schema/citation membership and review groundedness.

Next, optionally replace local retrieval with the inspected Knowledge Base from tutorial 10. Run the same fixtures before adding agent autonomy. Then host the application through the dedicated AgentCore runtime from tutorial 11 or a simpler approved API/compute path. Choose one path and record why it fits.

Finally, add Gateway/Identity/Memory only for an identified requirement. A diagram with more components is not evidence of a more capable system. Keep each change independently reversible.

## 5. Implement release and failure gates

Use a build that checks Python syntax, deterministic tests, configuration validation, dependency/security checks appropriate to the project, and the bounded AI evaluation set. Use short-lived CI identity and preserve the same artifact/configuration through promotion.

Block release on observed unauthorized data/tool access. Investigate material regressions in groundedness, abstention, latency, or cost. Label thresholds as your project's criteria. Store the prior artifact, model/prompt configuration, index configuration/version, and guardrail version needed for rollback.

Perform these drills: remove a source document, revoke a user, fail the order tool, select an unavailable model in the lab, and exceed a small local request limit. Verify a controlled response and an actionable signal. Do not test an unbounded paid request loop.

## 6. Operate and document

Create a dashboard/runbook using tutorial 15. Record a sanitized trace showing authenticated tenant → authorized retrieval → generation → citation validation → response. For tool use, add the schema/authorization decision and result category. Preserve no real secret or customer record in the portfolio.

Write a design record comparing the extractive baseline, a deterministic model workflow, and a bounded agent. Explain when the agent adds value and when it adds unnecessary latency or risk. Include a current cost estimate with idle, expected, and failed-retry scenarios.

## 7. Predictive-model companion project

Use tutorials 01–05 to create a separate tabular prediction pipeline: synthetic data → leakage-safe preparation → baseline/candidate training → held-out evaluation → versioned registry artifact → batch inference. Join delayed labels to predictions and test schema/feature drift. Keep training and generation evaluation distinct; a RAG quality score does not validate a fraud or forecasting model.

## Cleanup and completion

Remove the dedicated runtime, knowledge base/store, model endpoints, schedules, gateway/memory resources, synthetic S3 artifacts, roles, and logs according to ownership and retention. Verify each service separately. Keep local fixtures, source, sanitized evaluation results, and the final resource ledger.

The deliverable is a repository that another learner can understand without deploying it: setup, diagrams, code, fixtures, limitations, tests, measured outcomes, costs, and teardown. State which stages actually ran locally, which ran in your sandbox, and which remain design exercises. This distinction is part of sound engineering evidence.
