# Build a verifiable release with a constrained CI identity

[Delivery tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../13-devops-and-delivery.md)

**Outcome:** Test a tiny Lambda application, create a deterministic ZIP and release manifest, and optionally publish it through a protected GitHub Actions job using OIDC. Local Python work makes no AWS calls. Hosted CI may consume paid minutes/storage; optional publishing changes an existing dedicated Lambda function and incurs AWS usage when invoked. You need an established sandbox function/execution role and deployment permissions before enabling publishing.

## 1. Write the application and meaningful tests

Save `app.py`. The handler accepts only a bounded integer and computes a price in integer cents, avoiding floating-point money arithmetic.

```python
def handler(event, context):
    quantity = event.get("quantity")
    if type(quantity) is not int or not 1 <= quantity <= 100:
        raise ValueError("quantity must be an integer from 1 to 100")
    return {"quantity": quantity, "total_cents": quantity * 125, "release": "v1"}
```

Save `test_app.py`:

```python
import unittest
from app import handler


class PriceTests(unittest.TestCase):
    def test_price(self):
        self.assertEqual(handler({"quantity": 2}, None)["total_cents"], 250)

    def test_boundaries(self):
        self.assertEqual(handler({"quantity": 100}, None)["total_cents"], 12500)

    def test_rejects_invalid_values(self):
        for value in (0, 101, -1, True, "2", None):
            with self.subTest(value=value), self.assertRaises(ValueError):
                handler({"quantity": value}, None)


if __name__ == "__main__":
    unittest.main()
```

Run `python -m unittest -v`. Expect three passing tests. Change the multiplier to 126 and rerun; the price tests must fail. Restore 125. A test that merely repeats implementation syntax without checking a business invariant would not catch a meaningful change as clearly.

## 2. Package the exact bytes and record provenance

Save `build_release.py`. It validates syntax and runs tests before writing an artifact. Fixed ZIP metadata makes the same source bytes produce the same ZIP bytes; the manifest still records the actual interpreter version and source commit.

```python
import ast
import hashlib
import json
import os
import platform
import subprocess
import sys
import zipfile
from pathlib import Path

source = Path("app.py").read_bytes()
ast.parse(source, filename="app.py")
subprocess.run([sys.executable, "-m", "unittest", "-v"], check=True)
info = zipfile.ZipInfo("app.py", date_time=(1980, 1, 1, 0, 0, 0))
info.compress_type = zipfile.ZIP_DEFLATED
info.external_attr = 0o644 << 16
with zipfile.ZipFile("release.zip", "w") as archive:
    archive.writestr(info, source)
manifest = {
    "artifact": "release.zip",
    "sha256": hashlib.sha256(Path("release.zip").read_bytes()).hexdigest(),
    "source_sha256": hashlib.sha256(source).hexdigest(),
    "commit": os.getenv("GITHUB_SHA", "local-uncommitted"),
    "python": platform.python_version(),
}
Path("release-manifest.json").write_text(json.dumps(manifest, indent=2), encoding="utf-8")
print(json.dumps(manifest, indent=2))
```

Run twice and compare SHA-256 values. Do not commit the generated ZIP as a substitute for source history. Keep artifact storage access-controlled and immutable by digest/version. For third-party packages, lock dependencies, verify hashes, package compatible binaries, scan, and record an SBOM. A signature establishes who attested to bytes; it does not establish that those bytes are secure. ECR uses image digests for the analogous container artifact; CodeArtifact can distribute versioned packages.

## 3. Prepare OIDC trust without long-lived keys

For an optional publish job, create an AWS OIDC provider for GitHub according to the current official instructions and a dedicated role. Its trust policy should require the exact audience and intended repository/environment subject. GitHub has introduced immutable owner/repository IDs in subject formats for applicable repositories; inspect your repository's actual documented subject settings rather than assuming an older name-based example fits.

This is a template, not a ready-to-apply policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {"StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": "EXACT_REVIEWED_SUBJECT_FOR_SANDBOX_ENVIRONMENT"
    }}
  }]
}
```

Allow that role only the target function operations required here, principally `lambda:UpdateFunctionCode`; add read operations only for chosen verification. It does not need administrator access or unrestricted pass-role. Configure a protected `sandbox` GitHub environment with appropriate reviewers/branch restrictions and variables `AWS_ROLE_ARN`, `AWS_REGION`, `AWS_ACCOUNT_ID`, and `LAMBDA_FUNCTION`. The deployable function must use handler `app.handler` and a supported compatible Python runtime. Its execution role is separate from the CI role.

## 4. Create the workflow template

Save `.github/workflows/release.yml`. Replace each `REVIEWED_*_SHA` with the actual reviewed full commit SHA of that action; these placeholders intentionally prevent accidental execution before dependency review. Pin the runner/runtime according to your team's policy. Before enabling publishing, ensure all existing traffic invokes a stable published version or alias. `update-function-code --publish` also changes `$LATEST`, so unqualified invocations receive the new code immediately. The template leaves alias promotion to the next tutorial; that separation protects only appropriately qualified callers.

```yaml
name: release
on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      publish:
        type: boolean
        default: false
permissions:
  contents: read
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@REVIEWED_CHECKOUT_SHA
        with:
          persist-credentials: false
      - uses: actions/setup-python@REVIEWED_SETUP_PYTHON_SHA
        with:
          python-version: '3.13'
      - shell: pwsh
        run: |
          python build_release.py
          if ($LASTEXITCODE -ne 0) { throw 'Build failed' }
      - uses: actions/upload-artifact@REVIEWED_UPLOAD_ARTIFACT_SHA
        with:
          name: release
          path: |
            release.zip
            release-manifest.json
          retention-days: 7
  publish:
    if: github.event_name == 'workflow_dispatch' && inputs.publish
    needs: build
    runs-on: ubuntu-latest
    environment: sandbox
    concurrency:
      group: sandbox-publish
      cancel-in-progress: false
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@REVIEWED_DOWNLOAD_ARTIFACT_SHA
        with:
          name: release
      - uses: aws-actions/configure-aws-credentials@REVIEWED_AWS_CREDENTIALS_SHA
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          allowed-account-ids: ${{ vars.AWS_ACCOUNT_ID }}
      - shell: pwsh
        env:
          FUNCTION: ${{ vars.LAMBDA_FUNCTION }}
        run: |
          $manifest = Get-Content release-manifest.json -Raw | ConvertFrom-Json
          $actual = (Get-FileHash release.zip -Algorithm SHA256).Hash.ToLowerInvariant()
          if ($actual -ne $manifest.sha256) { throw 'Artifact hash mismatch' }
          aws lambda update-function-code --function-name $env:FUNCTION --zip-file fileb://release.zip --publish
          if ($LASTEXITCODE -ne 0) { throw 'Version publication failed' }
```

A pull request can run tests without AWS credentials. Do not switch this to privileged `pull_request_target` execution of untrusted code. Do not download an artifact from an unrelated untrusted run into the publish job. This workflow uses the same run's build dependency and a protected manual dispatch, but repository permissions and environment protections remain essential.

## 5. Verify failure and cloud boundaries

First run only the build job. Confirm the artifact contains exactly `app.py`, its hash matches, and the manifest commit matches the reviewed commit. Modify the artifact after building; the publish hash check should fail before AWS mutation. A negative trust test from an unapproved branch/environment should fail role assumption. Do not print the OIDC token to diagnose trust; compare sanitized claims/settings and CloudTrail evidence.

If optionally publishing, record function ARN, returned immutable version, code hash, source commit, CI run, and artifact digest. Publishing does not move an existing alias, but this update changes `$LATEST`; verify both the alias target and every caller's qualifier before claiming traffic is unchanged. If a job fails after a successful publish but before saving metadata, inspect versions instead of blindly publishing again.

For an AWS-native alternative, connect the approved source through CodeConnections, use CodePipeline to sequence source/build/approval/deploy, and use CodeBuild with a reviewed build image and this buildspec:

```yaml
version: 0.2
phases:
  build:
    commands:
      - python build_release.py
artifacts:
  files:
    - release.zip
    - release-manifest.json
```

The selected image must already provide compatible Python. CodeBuild's role needs only source/artifact/log permissions plus explicitly required build actions. CodeDeploy can manage supported deployments and alarm-based rollback. CodeCommit is again available to new customers according to AWS's reopening announcement; CodeCatalyst's new-customer availability differs and it is not a prerequisite. Check current lifecycle notices before choosing either.

## 6. Cleanup and explain

Disable/remove the dedicated workflow or its manual publish path, delete test artifacts according to retention, and remove the dedicated OIDC role/provider only if no other repository uses them. Delete the sandbox function and its retained logs/versions when the recovery tutorial is finished. Remove dedicated CodePipeline/CodeBuild connections, buckets, roles, and logs if you created the alternative. CI queues, repeated retries, build compute, artifact storage, and logs all contribute to cost.

Explain why building again during promotion changes the artifact guarantee, why trust conditions need both audience and subject, and why pipeline success does not prove production health.

Official references: [GitHub OIDC for AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws), [CodeBuild buildspec](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html), [Lambda versions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html), [CodeCommit reopening](https://aws.amazon.com/blogs/devops/aws-codecommit-returns-to-general-availability/).
