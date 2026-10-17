# Tutorial 5: turn a training request into a pipeline and a reviewable model version

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** local definition/release-gate preparation; optional **billable AWS** execution. **Prerequisites:** Tutorials [3](03-sagemaker-training-and-experiments.md) and [4](04-sagemaker-inference.md), an approved pipeline role, and their saved JSON artifacts. **Result:** a one-step managed training pipeline, an evaluation gate, and a registry version initially marked pending approval.

## 1. Separate orchestration, evidence, and deployment

A pipeline coordinates work and records dependencies. A registry records a model version and how to serve it. Neither proves the model is useful. Approval status is metadata that your deployment workflow must enforce; it is not a replacement for IAM or a guarantee that nobody can deploy a raw model artifact through another path.

This first pipeline contains one actual training step. Data preparation has already produced immutable, hashed artifacts. Evaluation and registration are explicit subsequent stages run by the learner, so no hidden script automatically promotes a model. After understanding this path, add Processing, Transform, Condition, and RegisterModel steps using the supported pipeline schema or pinned SDK. That extension requires validating the dataflow and job roles, not merely drawing extra boxes.

## 2. Build the pipeline definition locally

Save as `make_pipeline.py` beside `training-request.json`. It copies a valid request from Tutorial 3 and removes fields whose lifecycle belongs to the pipeline rather than the individual manual job.

```python
import copy
import json
from pathlib import Path

request = json.loads(Path("training-request.json").read_text(encoding="utf-8"))
arguments = copy.deepcopy(request)
arguments.pop("TrainingJobName", None)
arguments.pop("Tags", None)
arguments.pop("ExperimentConfig", None)
definition = {
    "Version": "2020-12-01",
    "Parameters": [],
    "Steps": [{"Name": "TrainRoutes", "Type": "Training", "Arguments": arguments}],
}
assert arguments["ResourceConfig"]["InstanceCount"] == 1
assert arguments["StoppingCondition"]["MaxRuntimeInSeconds"] <= 300
Path("pipeline-definition.json").write_text(json.dumps(definition, indent=2), encoding="utf-8")
print("Prepared one Training step; no AWS calls made")
```

Run `python .\make_pipeline.py`. Inspect all S3 URIs, image, training role, instance type, and runtime cap. They must reference retained inputs from a prior **submitted** run; a dry-run request may name objects that were never uploaded. The pipeline's generated training job name differs from the earlier manual job.

The raw JSON format is documented in [Define a pipeline](https://docs.aws.amazon.com/sagemaker/latest/dg/define-pipeline.html). Using low-level JSON makes the data contract visible, but it means you must validate against the current service rather than assuming an SDK silently corrects it.

## 3. Optionally create and execute it once

The pipeline role must be able to start/describe training and pass the approved training role. Its trust policy allows SageMaker to assume it. The training role still needs its own S3/container/logging access. Restrict both roles to the intended resources; one role's permissions do not automatically transfer to the other.

Set `SM_PIPELINE_ROLE` and a new `SM_PIPELINE_NAME`, then use this submission block only after checking account identity and current cost. It assumes `AWS_PROFILE`, `AWS_REGION`, and `EXPECTED_AWS_ACCOUNT` from Tutorial 3.

```python
import json
import os
from pathlib import Path
import boto3

session = boto3.Session(region_name=os.environ["AWS_REGION"])
if session.client("sts").get_caller_identity()["Account"] != os.environ["EXPECTED_AWS_ACCOUNT"]:
    raise RuntimeError("Unexpected AWS account")
sm = session.client("sagemaker")
name = os.environ["SM_PIPELINE_NAME"]
created = sm.create_pipeline(
    PipelineName=name,
    RoleArn=os.environ["SM_PIPELINE_ROLE"],
    PipelineDefinition=Path("pipeline-definition.json").read_text(encoding="utf-8"),
)
Path("pipeline-created.json").write_text(json.dumps({"name": name,
    "arn": created["PipelineArn"]}, indent=2), encoding="utf-8")
started = sm.start_pipeline_execution(PipelineName=name)
Path("pipeline-execution.json").write_text(json.dumps({"name": name,
    "execution": started["PipelineExecutionArn"]}, indent=2), encoding="utf-8")
print(started["PipelineExecutionArn"])
```

Observe the execution through `describe-pipeline-execution` and `list-pipeline-execution-steps` with its recorded ARN. Verify the training step's job ARN, then inspect that job and its output. A pipeline created successfully but never started has performed no training. A failed step is not permission to register an older artifact under the new run's name.

Use the new completed training description to repeat Tutorial 4 and produce an evaluation tied to the pipeline run. First archive the manual run's files and preserve its `training-run.json` as `training-run-manual.json`. Save the pipeline step's completed job description as the new `training-description.json`. Tutorial 4 deliberately rejects mismatched job names, so update the local run record explicitly with the following local-only snippet before invoking it. The pipeline reused the same input objects, image, and role; verify those match the new description before proceeding.

```python
import json
from pathlib import Path

run = json.loads(Path("training-run-manual.json").read_text(encoding="utf-8"))
trained = json.loads(Path("training-description.json").read_text(encoding="utf-8-sig"))
if trained["TrainingJobStatus"] != "Completed":
    raise ValueError("Pipeline training job has not completed")
if trained["AlgorithmSpecification"]["TrainingImage"] != run["request"]["AlgorithmSpecification"]["TrainingImage"]:
    raise ValueError("Training image differs from the recorded request")
if trained["RoleArn"] != run["request"]["RoleArn"]:
    raise ValueError("Training role differs from the recorded request")
run["job"] = trained["TrainingJobName"]
run["job_arn"] = trained["TrainingJobArn"]
run["request"]["TrainingJobName"] = run["job"]
run["pipeline_execution"] = json.loads(Path("pipeline-execution.json").read_text(encoding="utf-8"))["execution"]
Path("training-run.json").write_text(json.dumps(run, indent=2), encoding="utf-8")
Path("training-request.json").write_text(json.dumps(run["request"], indent=2), encoding="utf-8")
```

Record preprocessing hash, training job, model artifact, evaluation input hash, and report together. This is your lineage chain. Preserve the manual and pipeline run directories separately so later experiments do not overwrite their evidence.

## 4. Build a deterministic release gate

Save as `release_gate.py`. The threshold is a teaching choice declared before inspecting results, not a universal model-quality standard. It checks evaluation counts and manifest presence as well as accuracy. A real gate also needs segment performance, privacy, operational limits, and business risk criteria.

```python
import argparse
import json
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("--minimum-accuracy", type=float, default=0.6)
args = parser.parse_args()
if not 0 <= args.minimum_accuracy <= 1:
    raise ValueError("Accuracy threshold must be between 0 and 1")
report = json.loads(Path("evaluation.json").read_text(encoding="utf-8"))
manifest = json.loads(Path("data/manifest.json").read_text(encoding="utf-8"))
assert report["count"] == 20
assert sum(report[k] for k in ("tp", "tn", "fp", "fn")) == report["count"]
passed = report["accuracy"] >= args.minimum_accuracy
decision = {"passed": passed, "threshold": args.minimum_accuracy,
            "accuracy": report["accuracy"], "input_hash": manifest["inference.csv"]}
Path("release-decision.json").write_text(json.dumps(decision, indent=2), encoding="utf-8")
print(json.dumps(decision))
raise SystemExit(0 if passed else 1)
```

Run it and check the exit code. For the controlled failure, copy the report to a scratch directory, lower its accuracy below the declared threshold, and run the same gate there. Expect exit code 1 and `passed: false`. Restore the original report; never silently alter evidence to pass a gate.

## 5. Register an evaluated version as pending

After a passing decision and a manual review of lineage, create a uniquely named model package group using `create_model_package_group(ModelPackageGroupName=...)`. Then call the following request with your trained artifact and image. This is an AWS mutation; use the same checked account/session and retain the resulting ARN.

```python
decision = json.loads(Path("release-decision.json").read_text(encoding="utf-8"))
if not decision["passed"]:
    raise RuntimeError("Release gate failed")
trained = json.loads(Path("training-description.json").read_text(encoding="utf-8-sig"))
request = json.loads(Path("training-request.json").read_text(encoding="utf-8"))
package = sm.create_model_package(
    ModelPackageGroupName=os.environ["SM_PACKAGE_GROUP"],
    ModelPackageDescription="Synthetic routes; reviewed lineage and held-out evaluation",
    ModelApprovalStatus="PendingManualApproval",
    InferenceSpecification={
        "Containers": [{"Image": request["AlgorithmSpecification"]["TrainingImage"],
                        "ModelDataUrl": trained["ModelArtifacts"]["S3ModelArtifacts"]}],
        "SupportedContentTypes": ["text/csv"],
        "SupportedResponseMIMETypes": ["text/csv"],
    },
)
Path("registered-model.json").write_text(json.dumps({"arn": package["ModelPackageArn"]},
    indent=2), encoding="utf-8")
```

Check package status and approval state through `describe-model-package`. Do not mark it Approved merely to finish a tutorial. Write the actual deployment decision and who owns it. In a complete delivery workflow, the deployment role should only be used through a path that checks approved package ARN, artifact identity, and required evaluation evidence.

## Troubleshooting and operational extensions

Pipeline schema validation errors occur before useful work; inspect the exact step/argument. A training access denial can originate in the pipeline role's `PassRole` permissions or the training role's data access. A completed pipeline with an empty registry is expected here because registration is an explicit later stage. A package stuck in a failed state requires reading its failure reason rather than creating many duplicate versions.

For an advanced extension, add a Processing step that writes an evaluation JSON, a PropertyFile reference, and a Condition step that routes pass/fail outcomes. Compare two artifacts through a controlled canary or shadow evaluation. Monitor delayed-label quality with scheduled jobs and CloudWatch; do not assume maintenance-stage Model Monitor is available to new customers. Define rollback as a return to a previously approved package plus compatible preprocessing, not just an old container tag.

## Cost and teardown

Pipeline executions incur their underlying job costs; retries and repeated experiments multiply them. Registry records do not replace S3 retention planning. Stop a running pipeline execution using its exact ARN if abandoning it, then verify child jobs' states. Delete the dedicated pipeline once no execution is running. Remove only lab package versions before deleting their dedicated group; first ensure no endpoint or model depends on them. Delete dedicated models/endpoints from Tutorial 4 separately, then remove retained S3 artifacts, evaluation reports, and logs when no longer needed.

Explain: What is the pipeline role allowed to do? What connects a report to a particular model? Why is “Approved” not an IAM boundary? Which preprocessing version must roll back with a model? What should happen when labels arrive weeks after predictions?

## Sources

- [Pipeline step types](https://docs.aws.amazon.com/sagemaker/latest/dg/build-and-manage-steps-types.html).
- [CreatePipeline](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreatePipeline.html).
- [CreateModelPackage](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateModelPackage.html).
- [Model Monitor availability and alternatives](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor-availability-change.html).

Reviewed 2026-10-18. The default definition/gate work is local; cloud execution and registration require the stated optional steps.
