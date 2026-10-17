# Tutorial 4: run batch inference and evaluate a saved model

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** local evaluation plus optional **billable SageMaker batch transform**. **Prerequisites:** the artifacts from [Tutorial 2](02-data-preparation-and-feature-engineering.md) and a completed [Tutorial 3](03-sagemaker-training-and-experiments.md) job. **Result:** twenty predictions linked to held-out labels, measured errors, and a serving-mode decision. This path creates no continuously running endpoint.

## 1. Choose the serving mode

Batch transform is appropriate for our finite file of routes. Real-time inference serves interactive requests and normally keeps provisioned capacity available. Asynchronous inference accepts supported larger/longer requests for later completion. Serverless inference has a separate compatibility, scaling, and cold-start profile. These are different products with constraints, not four spellings of the same deployment.

For this exercise, startup latency is acceptable and keeping an idle endpoint would add no educational value. A model resource records the container and artifact to use; it is not itself a running endpoint. The transform job supplies temporary processing capacity. See [AWS inference options](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html).

## 2. Validate the model interface locally

Open `data/preprocessing.json`: six features in a fixed order. Confirm `data/inference.csv` contains twenty rows with six numeric values each. The label is absent. It would be a serious evaluation error to send the label as a seventh feature or change the category mapping after training.

Save this as `check_inputs.py`:

```python
import csv
import json
import math
from pathlib import Path

state = json.loads(Path("data/preprocessing.json").read_text(encoding="utf-8"))
rows = list(csv.reader(Path("data/inference.csv").open(encoding="utf-8")))
labels = json.loads(Path("data/labels.json").read_text(encoding="utf-8"))
assert len(rows) == len(labels) == 20, "row count mismatch"
for row in rows:
    assert len(row) == len(state["feature_order"]), "feature count mismatch"
    assert all(math.isfinite(float(value)) for value in row), "invalid numeric input"
print("20 rows; 6 finite numeric features; labels held separately")
```

Run `python .\check_inputs.py` and check its exit code. This checks shape, not semantic correctness: swapping two numeric columns could still pass. Compare a transformed record with its raw source and saved feature order too.

## 3. Create a model and one transform job

Reuse Tutorial 3's approved environment variables and save its completed job description as `training-description.json`. Set `SM_INFERENCE_INSTANCE` to a supported CPU instance compatible with your XGBoost image and batch-transform quota. This script defaults to local request preparation; `--submit` creates AWS resources.

```python
import argparse
import json
import os
import uuid
from pathlib import Path


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--submit", action="store_true")
    args = parser.parse_args()
    run = json.loads(Path("training-run.json").read_text(encoding="utf-8"))
    trained = json.loads(Path("training-description.json").read_text(encoding="utf-8-sig"))
    if trained["TrainingJobStatus"] != "Completed":
        raise ValueError("Training is not completed")
    if trained["TrainingJobName"] != run["job"]:
        raise ValueError("Description belongs to another training run")
    name = "roadmap-infer-" + uuid.uuid4().hex[:12]
    bucket, prefix = run["bucket"], run["prefix"] + "/" + name
    model = {
        "ModelName": name,
        "ExecutionRoleArn": run["request"]["RoleArn"],
        "PrimaryContainer": {
            "Image": run["request"]["AlgorithmSpecification"]["TrainingImage"],
            "ModelDataUrl": trained["ModelArtifacts"]["S3ModelArtifacts"],
        },
    }
    transform = {
        "TransformJobName": name, "ModelName": name,
        "BatchStrategy": "SingleRecord", "MaxConcurrentTransforms": 1,
        "TransformInput": {
            "DataSource": {"S3DataSource": {"S3DataType": "S3Prefix",
                "S3Uri": f"s3://{bucket}/{prefix}/input/inference.csv"}},
            "ContentType": "text/csv", "SplitType": "Line",
        },
        "TransformOutput": {"S3OutputPath": f"s3://{bucket}/{prefix}/output/",
                            "Accept": "text/csv", "AssembleWith": "Line"},
        "TransformResources": {"InstanceType": os.environ["SM_INFERENCE_INSTANCE"],
                               "InstanceCount": 1},
    }
    record = {"name": name, "bucket": bucket, "prefix": prefix,
              "model": model, "transform": transform}
    Path("inference-run.json").write_text(json.dumps(record, indent=2), encoding="utf-8")
    if args.submit:
        import boto3
        session = boto3.Session(region_name=run["region"])
        if session.client("sts").get_caller_identity()["Account"] != os.environ["EXPECTED_AWS_ACCOUNT"]:
            raise RuntimeError("Unexpected account")
        session.client("s3").upload_file("data/inference.csv", bucket,
                                         prefix + "/input/inference.csv")
        sm = session.client("sagemaker")
        sm.create_model(**model)
        sm.create_transform_job(**transform)
    print(name)


if __name__ == "__main__":
    main()
```

Save as `batch_infer.py`; run it without flags to inspect `inference-run.json`. Recheck the image, artifact URI, role, and instance cost. Run once with `--submit` only when ready. The record is written before mutations so a partial failure leaves the exact model name available for cleanup.

## 4. Observe, download, and score

Use the recorded name with `aws sagemaker describe-transform-job --transform-job-name <name>` and your explicit profile/Region. Poll at a bounded interval. On `Failed`, inspect `FailureReason` and the transform logs; do not blindly restart. On `Completed`, download the exact output object from `TransformOutput.S3OutputPath` plus `inference.csv.out` into `predictions.out`.

The single input file preserves record order in its corresponding output file. For a real pipeline, retain explicit identifiers and manifests rather than joining unrelated files by incidental listing order. Save this scoring program as `score.py`:

```python
import json
import math
from pathlib import Path

labels = json.loads(Path("data/labels.json").read_text(encoding="utf-8"))
scores = [float(line.strip()) for line in
          Path("predictions.out").read_text(encoding="utf-8").splitlines() if line.strip()]
assert len(scores) == len(labels), "prediction count mismatch"
assert all(math.isfinite(p) and 0 <= p <= 1 for p in scores), "invalid score"
counts = {"tp": 0, "tn": 0, "fp": 0, "fn": 0}
errors = []
for row, score in zip(labels, scores):
    actual, predicted = row["late"], int(score >= 0.5)
    key = ("tp" if actual else "fp") if predicted else ("fn" if actual else "tn")
    counts[key] += 1
    if actual != predicted:
        errors.append({"id": row["id"], "actual": actual, "score": score})
report = {"threshold": 0.5, "count": len(scores), **counts, "errors": errors,
          "accuracy": (counts["tp"] + counts["tn"]) / len(scores)}
Path("evaluation.json").write_text(json.dumps(report, indent=2), encoding="utf-8")
print(json.dumps(report, indent=2))
```

The expected invariant is twenty finite probabilities and counts summing to twenty. Model accuracy is measured, not predetermined. The threshold here was fixed in advance; if you tune it using these test outcomes, this set is no longer untouched evaluation data.

## 5. Controlled failure and endpoint extension

For a local failure drill, remove one line from a **copy** of `predictions.out`; the scorer must fail on count mismatch. Restore the original. Try adding a nonnumeric inference column in a temporary input copy; `check_inputs.py` should reject it before cloud submission.

To explore real-time hosting, use the same model to create an endpoint configuration with one production variant, a chosen instance type/count, and initial weight 1; then create an endpoint referencing that configuration. Wait for `InService`, call `sagemaker-runtime.invoke_endpoint` with `ContentType="text/csv"` and **one six-feature row**, and compare the score to batch output. Use the official [CreateEndpointConfig](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateEndpointConfig.html), [CreateEndpoint](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateEndpoint.html), and [InvokeEndpoint](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_runtime_InvokeEndpoint.html) request examples with your recorded model name. This is optional and begins ongoing endpoint charges; do not substitute unsupported instance/serverless settings.

For async/serverless, first produce a comparison worksheet: payload/latency needs, cold-start tolerance, model size, supported features, scaling minimum, quotas, and total idle/busy cost. Implement only a documented compatible mode. GPU serving, multi-model hosting, and inference components are later extensions requiring their own capacity and model-isolation tests.

## Troubleshooting and teardown

Wrong feature count: compare training and inference column order. `ModelError`: inspect container logs and content type. S3 access denial: inspect the execution role and encryption key. Missing output: verify job status and exact object key; successful request submission is insufficient. A timed-out local poll does not stop a running job.

**Cost:** batch compute, S3 requests/storage, and logs; optional endpoints add capacity charges while idle. Stop an unwanted running transform job with `stop-transform-job`. After it finishes, delete the dedicated model with `delete-model`; this does not delete its S3 artifact. If you created an endpoint, delete the endpoint first, verify deletion, then its endpoint configuration and model. Inspect and remove only the recorded input/output objects and retained versions when no longer needed.

Explain: Why does batch avoid an idle endpoint? Which identity reads the artifact? How could valid numeric input still be wrong? What does deleting a model leave behind? Continue with [pipelines and registry](05-mlops-pipelines-and-model-registry.md).

## Sources

- [CreateTransformJob](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTransformJob.html).
- [Batch transform behavior](https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html).
- [CreateModel](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateModel.html).

Reviewed 2026-10-18. Cloud execution depends on current image, instance, permission, and quota compatibility.
