# Tutorial 1: train a small classifier and measure its mistakes

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** entirely local, standard-library Python. **Prerequisites:** Python 3, a text editor, and basic functions/lists. **Result:** a fitted decision stump, a held-out evaluation, and an experiment record you can reproduce. No AWS account, package installation, or personal data is required.

## 1. Define the decision before the model

Our fictional delivery team wants to predict whether a route will take more than its target duration. We will use a synthetic `distance` feature and a binary `late` label. This is supervised classification because examples contain the outcome we want to predict. A regression variant would predict the duration itself. Clustering would instead group similar routes without known late/on-time labels. Forecasting would predict a future time series and requires time-aware evaluation. Ranking would order routes by an operational priority rather than necessarily assigning a class.

Write this contract in `notes.md`: prediction occurs before departure; only distance is available; missing distance is rejected; a missed late delivery is a false negative; an unnecessary warning is a false positive. A field such as actual arrival time would leak the answer because it is unavailable at prediction time.

A threshold rule is a useful baseline. We will **learn** its threshold from training examples, compare it with a constant predictor on validation examples, then inspect the test split only after choosing. This makes training, model selection, and evaluation visible without hiding them behind a library.

## 2. Create an isolated working directory

Use a new directory of your choice. Save the following program as `foundations.py` in it. Run commands from that directory. The program writes only `experiment.json` beside itself; inspect that path before rerunning if you already have a file with that name.

```python
import json
from pathlib import Path


def metrics(rows, predict):
    counts = {"tp": 0, "tn": 0, "fp": 0, "fn": 0}
    for row in rows:
        actual = row["late"]
        guessed = predict(row["distance"])
        name = ("tp" if actual else "fp") if guessed else (
            "fn" if actual else "tn")
        counts[name] += 1
    tp, tn, fp, fn = (counts[k] for k in ("tp", "tn", "fp", "fn"))
    precision = tp / (tp + fp) if tp + fp else 0.0
    recall = tp / (tp + fn) if tp + fn else 0.0
    return {
        **counts,
        "accuracy": (tp + tn) / len(rows),
        "precision": precision,
        "recall": recall,
        "f1": 2 * precision * recall / (precision + recall)
        if precision + recall else 0.0,
    }


def predict_threshold(value, threshold):
    if type(value) not in (int, float) or not 0 <= value <= 100:
        raise ValueError("distance must be a number between 0 and 100")
    return int(value >= threshold)


def run():
    rows = [{"id": i, "distance": i, "late": int(i >= 19)}
            for i in range(1, 31)]
    rows[19]["late"] = 0  # One noisy exception at distance 20.
    train = [r for r in rows if r["id"] % 5 < 3]
    validation = [r for r in rows if r["id"] % 5 == 3]
    test = [r for r in rows if r["id"] % 5 == 4]
    assert not ({r["id"] for r in train} & {r["id"] for r in test})

    candidates = range(0, 32)
    threshold = max(candidates, key=lambda t: (
        metrics(train, lambda x: predict_threshold(x, t))["f1"], -t))
    majority = int(sum(r["late"] for r in train) > len(train) / 2)
    models = {
        "constant": lambda x: majority,
        "threshold": lambda x: predict_threshold(x, threshold),
    }
    validation_scores = {name: metrics(validation, fn)
                         for name, fn in models.items()}
    chosen = max(models, key=lambda name: validation_scores[name]["f1"])
    report = {
        "data_version": "synthetic-routes-v1",
        "split_sizes": [len(train), len(validation), len(test)],
        "threshold": threshold,
        "validation": validation_scores,
        "selected": chosen,
        "test": metrics(test, models[chosen]),
    }
    Path("experiment.json").write_text(json.dumps(report, indent=2),
                                        encoding="utf-8")
    print(json.dumps(report, indent=2))


if __name__ == "__main__":
    run()
```

```powershell
python .\foundations.py
if ($LASTEXITCODE -ne 0) { throw 'The local experiment failed.' }
Get-Content -LiteralPath .\experiment.json
```

## 3. Interpret the evidence

The split sizes should be `18, 6, 6`. The selected model should be `threshold`, and the learned threshold is `21`. The validation split is perfectly classified, but the test split contains a late distance-19 route that becomes a false negative. Expect test counts `tp=2`, `tn=3`, `fp=0`, `fn=1`, accuracy about `0.8333`, and F1 `0.8`. This is a result on six synthetic cases, not a production accuracy claim.

Check each confusion-matrix cell by hand. A true positive is actually late and predicted late. Precision asks how many warnings were correct; recall asks how many late deliveries were detected. F1 balances those two quantities but does not encode every business cost. A constant on-time prediction may look reasonable under extreme imbalance while missing every late delivery.

Our score is a binary class, **not a calibrated probability**. Calling a distance-derived number “80% confidence” would require a separate probability model and calibration assessment. For a real classifier, inspect thresholds and a precision-recall curve; accuracy alone is particularly weak when the positive outcome is rare. For regression, compare MAE and RMSE and inspect large-error examples rather than converting regression error into classification accuracy.

## 4. Run a controlled failure and a shift experiment

Use this local command to exercise the input contract:

```powershell
python -c "from foundations import predict_threshold; predict_threshold(-1, 18)"
```

Expected: a `ValueError` and a nonzero process exit. That failure is intentional. Restore normal behavior by passing a valid numeric distance. Do not make the function silently convert missing values to zero; that would turn missing data into a plausible on-time route.

Next, copy the test rows into a separate diagnostic exercise and change their late condition to distance at least 10, representing severe traffic. Keep the original fitted threshold fixed. You should see false negatives increase. This is a distribution/relationship change, not proof the original training script malfunctioned. Record the changed assumption, resulting errors, and whether new features or retraining would actually address the cause.

Do not repeatedly adjust the model against the original test labels and continue calling that split “held out.” Once you use it for decisions, obtain a fresh evaluation set. For historical production data, split by time and possibly by customer/site so correlated examples do not leak across the evaluation boundary.

## 5. Troubleshoot, retain evidence, and clean up

If Python cannot find the file, inspect `Get-Location` and `Get-ChildItem`; avoid changing system directories. If a different interpreter runs, inspect `python --version`. If results differ, compare the exact code and synthetic-data version before attributing differences to randomness: this program contains no random sampling. If you changed the label rule, that is a new experiment and deserves a new record.

Save the data definition, feature availability time, chosen threshold, split policy, confusion matrices, and the test false-negative explanation. Add a short limitation statement: one feature, tiny synthetic data, no real-world generalization evidence, and no calibrated probabilities. These details are more valuable than a screenshot claiming a perfect model.

**Cost:** local CPU only; no billable AWS actions. **Teardown:** keep the source for learning or remove only `foundations.py` and `experiment.json` from this dedicated directory. No cloud resource cleanup is needed.

Explain before continuing: What is learned from training? What did validation select? Why can a perfect tiny test score be misleading? Which field would create leakage? How would the split change for weekly route data? What business cost does F1 omit? Continue with [data preparation](02-data-preparation-and-feature-engineering.md).

## Sources

- [SageMaker ML workflow concepts](https://docs.aws.amazon.com/sagemaker/latest/dg/how-it-works-mlconcepts.html) — AWS lifecycle context.
- [Scikit-learn model evaluation](https://scikit-learn.org/stable/modules/model_evaluation.html) — metric definitions and their limitations.
- [Scikit-learn common pitfalls](https://scikit-learn.org/stable/common_pitfalls.html) — leakage and inconsistent preprocessing.

Reviewed 2026-10-18. The local program is educational; it does not simulate SageMaker infrastructure.
