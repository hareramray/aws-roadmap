# Tutorial 2: build a reproducible, leakage-safe training dataset

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** local Python standard library. **Prerequisite:** [Tutorial 1](01-ml-foundations.md). **Result:** synthetic raw data, an explicit feature contract, training/validation files, held-out inference inputs and labels, and a preprocessing manifest. Later SageMaker tutorials use these exact artifacts.

## 1. Define the dataset contract

Each record represents a synthetic delivery with `id`, `event_day`, `distance`, `stops`, `area`, and `late`. The prediction is made before departure, so these features are allowed. The label describes the eventual outcome and is never included in inference input. We deliberately include missing distances and a later unseen area to exercise transformation behavior.

The split is chronological: first 60 records for training, next 20 for validation, last 20 for testing. Time order is illustrative rather than a claim that one hundred records establish a reliable production model. Preprocessing statistics and category vocabulary come from training only. Validation helps choose settings; testing stays separate until evaluation.

The output uses numeric CSV with the label in the first column for the SageMaker built-in XGBoost algorithm. There is no header. Inference CSV contains the exact same feature columns in the same order but omits the label. Feature order is part of the model interface, not an incidental spreadsheet layout.

## 2. Generate, validate, and transform

Create a dedicated local directory and save this as `prepare.py`. It writes under `data/` only. Do not place unrelated files in that directory. The `--bad` switch creates a controlled invalid record before publication.

```python
import argparse
import csv
import hashlib
import json
import statistics
from pathlib import Path


def make_rows():
    rows = []
    for i in range(100):
        actual_distance = 1 + (i * 7) % 30
        stops = (i * 3) % 8
        rows.append({
            "id": f"route-{i:03d}", "event_day": i,
            "distance": None if i % 13 == 0 else actual_distance,
            "stops": stops,
            "area": "rural" if i >= 90 else ("urban" if i % 2 else "town"),
            "late": int(actual_distance + 2 * stops > 25),
        })
    return rows


def validate(rows):
    seen = set()
    for row in rows:
        if row["id"] in seen:
            raise ValueError("duplicate business key")
        seen.add(row["id"])
        distance = row["distance"]
        if distance is not None and not 0 <= distance <= 100:
            raise ValueError(f"invalid distance for {row['id']}")
        if row["late"] not in (0, 1):
            raise ValueError("label must be binary")
        if type(row["stops"]) is not int or not 0 <= row["stops"] <= 20:
            raise ValueError("invalid stops")


def transform(row, state):
    missing = row["distance"] is None
    value = state["distance_median"] if missing else row["distance"]
    known = state["areas"]
    return [value, row["stops"], int(missing),
            *[int(row["area"] == area) for area in known],
            int(row["area"] not in known)]


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--bad", action="store_true")
    args = parser.parse_args()
    rows = make_rows()
    if args.bad:
        rows[3]["distance"] = -10
    validate(rows)  # Fail before overwriting any output.
    train, valid, test = rows[:60], rows[60:80], rows[80:]
    state = {
        "schema_version": 1,
        "distance_median": statistics.median(
            r["distance"] for r in train if r["distance"] is not None),
        "areas": sorted({r["area"] for r in train}),
        "feature_order": ["distance", "stops", "distance_missing",
                          "area_town", "area_urban", "area_unknown"],
    }
    output = Path("data")
    output.mkdir(exist_ok=True)
    (output / "raw.json").write_text(json.dumps(rows, indent=2), encoding="utf-8")
    (output / "preprocessing.json").write_text(
        json.dumps(state, indent=2), encoding="utf-8")
    for name, part in (("train", train), ("validation", valid), ("test", test)):
        with (output / f"{name}.csv").open("w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f, lineterminator="\n")
            writer.writerows([[r["late"], *transform(r, state)] for r in part])
    with (output / "inference.csv").open("w", newline="", encoding="utf-8") as f:
        csv.writer(f, lineterminator="\n").writerows(transform(r, state) for r in test)
    labels = [{"id": r["id"], "late": r["late"]} for r in test]
    (output / "labels.json").write_text(json.dumps(labels), encoding="utf-8")
    files = ["raw.json", "preprocessing.json", "train.csv", "validation.csv",
             "test.csv", "inference.csv", "labels.json"]
    manifest = {name: hashlib.sha256((output / name).read_bytes()).hexdigest()
                for name in files}
    (output / "manifest.json").write_text(json.dumps(manifest, indent=2), encoding="utf-8")
    assert len(transform(test[-1], state)) == len(state["feature_order"])
    assert transform(test[-1], state)[-1] == 1
    print(json.dumps({"rows": [60, 20, 20], "features": state["feature_order"],
                      "unseen_test_area": test[-1]["area"]}, indent=2))


if __name__ == "__main__":
    main()
```

```powershell
python .\prepare.py
if ($LASTEXITCODE -ne 0) { throw 'Preparation failed.' }
Get-Content -LiteralPath .\data\preprocessing.json
Get-Content -LiteralPath .\data\train.csv -TotalCount 3
```

## 3. Verify the contract, not just file existence

Expected: 60 training, 20 validation, and 20 test records; six feature names; `rural` reported as the unseen test area. Training CSV rows have seven values: label plus six features. Inference rows have six values. The missing-distance indicator distinguishes an imputed value from a measured value equal to the median.

The `area_unknown` column is essential. Silently extending the vocabulary during inference changes the vector length and can break or corrupt predictions. Unknown categories may be handled differently in a real model, but the behavior must be explicit and evaluated. Tree models generally do not require numeric standardization. Linear models and distance-based methods often benefit from scaling; fit scaling parameters on the training split only and save them alongside this manifest.

Run the program twice and compare `manifest.json`: the hashes should be unchanged. Hashes prove byte identity, not data correctness or privacy. Validate the row counts and a few transformed records by hand as separate checks.

## 4. Demonstrate a controlled data failure

```powershell
python .\prepare.py --bad
$BadRunExit = $LASTEXITCODE
if ($BadRunExit -eq 0) { throw 'The invalid-distance test should have failed.' }
```

Expected: a `ValueError` naming `route-003`. Validation occurs before writing files, so a previously successful dataset remains intact. That is a small local publication guarantee; a multi-file production pipeline needs a staging location plus a manifest or transactional commit so readers cannot observe a half-published dataset.

For a duplicate test, append `rows[0].copy()` before `validate(rows)` in a temporary copy of the script. It must fail. Decide whether a real source duplicate represents retry, correction, or conflicting data before choosing “keep first” or “keep last.” Business keys and source sequence numbers matter more than file arrival order.

## 5. Connect the example to real feature engineering

For time-dependent features, compute a feature using information available at the prediction timestamp. A customer's final monthly spend cannot be used to predict an event that happened halfway through that month. Offline feature retrieval needs point-in-time joins, while online serving needs freshness limits and missing-value behavior. A feature store can coordinate definitions and access, but it cannot detect every causal leakage error for you.

Keep labels, access permissions, consent, dataset licenses, retention, and lineage explicit. For human labeling, write annotation instructions, resolve disagreements, sample for quality, and protect the data presented to annotators. Do not assume maintenance-stage labeling products are available to a new AWS account; choose a currently supported workflow after checking the lifecycle register.

## Troubleshooting, costs, and teardown

If a future model reports the wrong feature count, compare `preprocessing.json` with its training version before editing input. If inference accuracy falls while the shape is valid, check swapped columns, changed units, missing-value rates, and unseen-category frequency. If the median changes between otherwise identical runs, verify that the training split and source bytes are actually identical.

**Cost:** no AWS charges in this tutorial. Uploading these artifacts in the next tutorial can create S3 request/storage charges and training charges. **Cleanup:** retain `data/` for Tutorials 3–5. If deleting it, inspect the directory and remove only this tutorial's named files; do not recursively remove a shared dataset folder.

Explain: Why is the label omitted from inference? Why is the median fitted on training only? What does the unknown-category feature preserve? What does a hash fail to prove? How would a source correction affect the manifest and model lineage?

## Sources

- [SageMaker XGBoost input/output interface](https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost.html).
- [SageMaker Feature Store](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html).
- [Scikit-learn preprocessing pitfalls](https://scikit-learn.org/stable/common_pitfalls.html).
- [AWS feature lifecycle](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).

Reviewed 2026-10-18. The example prepares data locally and does not create a managed feature store.
