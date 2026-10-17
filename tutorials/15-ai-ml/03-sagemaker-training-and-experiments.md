# Tutorial 3: submit one bounded SageMaker training job

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** local request preparation first; optional **billable AWS** training. **Prerequisite:** [prepared data](02-data-preparation-and-feature-engineering.md), CLI v2, an authorized sandbox, and a reviewed training role. **Result:** a small XGBoost classifier plus a reproducible experiment record. No endpoint is created.

## 1. Understand the job boundary

SageMaker training launches managed compute, downloads input channels, runs the selected container, and uploads model artifacts. Your application does not keep a permanent training instance. You still own input correctness, the container version, permissions, resources, runtime limits, and the interpretation of results.

We use **built-in algorithm mode**, not framework script mode. XGBoost receives numeric CSV with the label first; `train` and `validation` are separate channels. Do not put test data into either channel. A framework container expecting your script is a different contract and cannot be substituted merely because its name contains “XGBoost.” The [XGBoost guide](https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost.html) links the supported versions and regional container registry paths.

## 2. Prepare account, permissions, and configuration

Use an existing private sandbox bucket in the same Region. The execution role must trust `sagemaker.amazonaws.com` and permit access to the exact input/output prefixes, required container pulls/logging, and applicable encryption keys. Your human/deployment role needs job operations and narrowly scoped `iam:PassRole` for that execution role. Do not create a broad role merely to bypass a failure.

Choose a currently supported **algorithm-mode CPU image** from AWS's region-specific registry reference and a compatible small CPU training instance supported by your sandbox quota. Record the image tag; do not use `latest`. Set the following variables after replacing every example value:

```powershell
$env:AWS_PROFILE = 'roadmap'
$env:AWS_REGION = 'ap-south-1'
$env:EXPECTED_AWS_ACCOUNT = 'REPLACE_WITH_12_DIGITS'
$env:SM_BUCKET = 'REPLACE_WITH_YOUR_PRIVATE_BUCKET'
$env:SM_EXECUTION_ROLE = 'REPLACE_WITH_APPROVED_ROLE_ARN'
$env:SM_TRAINING_IMAGE = 'REPLACE_WITH_REGION_SPECIFIC_ALGORITHM_IMAGE_URI'
$env:SM_INSTANCE_TYPE = 'REPLACE_WITH_SUPPORTED_CPU_INSTANCE_TYPE'
aws sts get-caller-identity --profile $env:AWS_PROFILE --region $env:AWS_REGION
```

The identity call is an optional AWS read; confirm its account before submission. Install a supported `boto3` into your own virtual environment only if you choose the AWS extension, then record its exact version in your project dependency lock. This guide does not require a SageMaker SDK major version because it uses the documented low-level API.

## 3. Generate a request without calling AWS

Save as `train_request.py` next to `data/`. The default invocation only reads local artifacts and writes local JSON. It deliberately rejects unset placeholders. Setting `--submit` is the point at which uploads and billable training begin.

```python
import argparse
import json
import os
import re
import uuid
from pathlib import Path


def required(name):
    value = os.environ.get(name, "")
    if not value or "REPLACE" in value:
        raise ValueError(f"Set {name} before continuing")
    return value


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--submit", action="store_true")
    args = parser.parse_args()
    region = required("AWS_REGION")
    bucket = required("SM_BUCKET")
    account = required("EXPECTED_AWS_ACCOUNT")
    if not re.fullmatch(r"\d{12}", account):
        raise ValueError("Expected account must have 12 digits")
    name = "roadmap-train-" + uuid.uuid4().hex[:12]
    prefix = "tutorials/" + name
    request = {
        "TrainingJobName": name,
        "RoleArn": required("SM_EXECUTION_ROLE"),
        "AlgorithmSpecification": {
            "TrainingImage": required("SM_TRAINING_IMAGE"),
            "TrainingInputMode": "File",
        },
        "HyperParameters": {"objective": "binary:logistic", "num_round": "15",
                            "max_depth": "2", "eta": "0.2", "eval_metric": "logloss",
                            "seed": "7"},
        "InputDataConfig": [
            {"ChannelName": channel, "ContentType": "text/csv",
             "DataSource": {"S3DataSource": {"S3DataType": "S3Prefix",
                 "S3Uri": f"s3://{bucket}/{prefix}/input/{channel}.csv",
                 "S3DataDistributionType": "FullyReplicated"}}}
            for channel in ("train", "validation")
        ],
        "OutputDataConfig": {"S3OutputPath": f"s3://{bucket}/{prefix}/output/"},
        "ResourceConfig": {"InstanceType": required("SM_INSTANCE_TYPE"),
                           "InstanceCount": 1, "VolumeSizeInGB": 10},
        "StoppingCondition": {"MaxRuntimeInSeconds": 300},
        "Tags": [{"Key": "Project", "Value": "aws-roadmap-tutorial"}],
    }
    manifest = json.loads(Path("data/manifest.json").read_text(encoding="utf-8"))
    record = {"job": name, "region": region, "bucket": bucket, "prefix": prefix,
              "dataset_hashes": manifest, "request": request, "submitted": False}
    Path("training-request.json").write_text(json.dumps(request, indent=2), encoding="utf-8")
    Path("training-run.json").write_text(json.dumps(record, indent=2), encoding="utf-8")
    if args.submit:
        import boto3
        session = boto3.Session(region_name=region)
        if session.client("sts").get_caller_identity()["Account"] != account:
            raise RuntimeError("Unexpected AWS account")
        s3 = session.client("s3")
        for channel in ("train", "validation"):
            s3.upload_file(f"data/{channel}.csv", bucket,
                           f"{prefix}/input/{channel}.csv")
        response = session.client("sagemaker").create_training_job(**request)
        record["submitted"] = True
        record["job_arn"] = response["TrainingJobArn"]
        Path("training-run.json").write_text(json.dumps(record, indent=2), encoding="utf-8")
    print(json.dumps({"job": name, "submitted": record["submitted"]}, indent=2))


if __name__ == "__main__":
    main()
```

```powershell
python .\train_request.py
Get-Content -LiteralPath .\training-request.json
```

Inspect the image, role, input URIs, runtime limit, and one-instance setting. The runtime cap limits training execution; it is not a universal spending cap covering storage, startup, or every related resource. Re-running generates a new name and record. Preserve a prior run record before starting another experiment.

## 4. Submit once and observe completion

After estimating regional charges and arranging quota, run:

```powershell
python .\train_request.py --submit
if ($LASTEXITCODE -ne 0) { throw 'Inspect training-run.json and AWS before retrying.' }
$TrainingRun = Get-Content -Raw -LiteralPath .\training-run.json | ConvertFrom-Json
aws sagemaker describe-training-job --training-job-name $TrainingRun.job --region $env:AWS_REGION --profile $env:AWS_PROFILE --output json --no-cli-pager
```

Poll that read at a deliberate interval, for example every 20 seconds for at most 15 minutes. Inspect `TrainingJobStatus`, `SecondaryStatus`, and `FailureReason`; do not assume a successful create response means training succeeded. A client timeout can occur after AWS accepted a job, so look up the recorded name before submitting another.

When status is `Completed`, save the description as `training-description.json`. Check `$LASTEXITCODE` before writing an API result to disk. The `ModelArtifacts.S3ModelArtifacts` value is the artifact URI used by Tutorial 4. Inspect `FinalMetricDataList` and training logs for validation log loss. Metrics are measured results; this tutorial does not promise an exact score across container versions.

## 5. Record an experiment and controlled failure

Keep the request, dataset hashes, image URI, SDK version, job ARN, runtime, validation metric, and training description together. If comparing two depths, change only `max_depth`, create a separate run record, and compare validation—not test—results. Managed MLflow or another approved tracker can store the same evidence; tracking is not a reason to leave a paid tracking environment running unnecessarily.

For a free failure drill, unset `SM_TRAINING_IMAGE` and run the default command. Expect a local `ValueError` before AWS access. Restore it afterward. For an already-approved cloud diagnostic exercise, use a nonexistent input key and inspect the job's specific failure; this can incur startup charges and should not be your first validation method.

Common failures: `AccessDenied` may concern your caller, `PassRole`, role trust, S3, or KMS; distinguish them using the exact failed action. `ResourceLimitExceeded` requires checking the correct SageMaker quota, not an EC2 instance quota. A CSV parsing failure often comes from a header, nonnumeric category, or wrong label position. Container failure requires a compatible image/instance combination and its logs.

## Cost, teardown, and questions

Training compute, attached job storage, S3, logs, and any configured networking can cost money. An active job can be stopped with `aws sagemaker stop-training-job --training-job-name <recorded-name>` using your explicit profile/Region. A completed job has no running training instance; its metadata is not a persistent endpoint.

Retain artifacts for the next tutorial or delete only the recorded input/output keys after inspecting their prefix. Account for object versions, logs, and any optional tracking server. No endpoint, custom KMS key, or new IAM role was created by the script.

Explain: Which role runs the container? Why is the test split absent? What does the create response prove? Why preserve an image version and data hashes? What remains billable after completion?

## Sources

- [CreateTrainingJob API](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTrainingJob.html).
- [XGBoost hyperparameters](https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost_hyperparameters.html).
- [Training image registry paths](https://docs.aws.amazon.com/sagemaker/latest/dg/ecr-us-east-1.html) — choose your Region in the documentation.
- [SageMaker experiment tracking with MLflow](https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow.html).

Reviewed 2026-10-18. AWS steps require actual account configuration and are not claimed to have been executed here.
