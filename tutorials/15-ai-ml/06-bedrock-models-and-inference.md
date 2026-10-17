# Tutorial 6: make one bounded Bedrock request and record its result

[AI tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../15-ai-ml-and-generative-ai.md)

**Mode:** local request preview; optional **billable Amazon Bedrock inference**. **Prerequisites:** Python, a supported Boto3 environment for the AWS extension, an authorized profile, and an eligible text model or inference profile. **Result:** one versioned request, a measured answer, token usage, and an explainable failure path. No model identifier is assumed to be available in every Region or account.

## 1. Choose a model by a concrete task

The task is to summarize a short synthetic incident in two sentences without inventing a cause. The input describes an API returning errors after a configuration deployment and recovery after rollback. A successful answer preserves what happened and the uncertainty about the underlying mechanism. We do not need tools, long memory, images, or a knowledge base.

In the current Bedrock model catalog, select a text model supporting the Converse API and the parameters used below. Verify model lifecycle, access/provider terms, Region, request limits, and the correct invocation identifier. Some models require an inference profile rather than direct on-demand invocation through the base model ID. A profile can route outside your source Region; inspect its geographic or global scope before sending sensitive content. This tutorial sends synthetic data only.

Use current [model access guidance](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) and [Converse compatibility](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-supported-models-features.html). Bedrock's managed infrastructure does not select an appropriate model or authorize your application users for you.

## 2. Configure the local experiment

Create a new directory and save `invoke_bedrock.py` there. Set the following variables with real sandbox values only when running the AWS extension:

```powershell
$env:AWS_PROFILE = 'roadmap'
$env:AWS_REGION = 'ap-south-1'
$env:EXPECTED_AWS_ACCOUNT = 'REPLACE_WITH_12_DIGITS'
$env:BEDROCK_MODEL_ID = 'REPLACE_WITH_SUPPORTED_MODEL_OR_PROFILE_ID'
```

The execution identity needs the relevant Bedrock invocation permission on the intended model/profile resources. Cross-Region profiles can require authorization for destination resources and compatible organizational policies. Do not resolve an access error with `bedrock:*` on every resource.

Record your Python/Boto3 versions and model selection in `notes.md`. For a reproducible project, lock the dependencies you actually tested rather than installing an unbounded version at every deployment. The preview path below does not import Boto3 or contact AWS.

## 3. Preview, then optionally invoke

```python
import argparse
import json
import os
import re
import time
from pathlib import Path


def request_body():
    return {
        "modelId": os.environ.get("BEDROCK_MODEL_ID", "CONFIGURE_MODEL_ID"),
        "system": [{"text": "Summarize only the supplied incident. Use two short sentences. "
                              "Do not invent a root cause. Distinguish observation from explanation."}],
        "messages": [{"role": "user", "content": [{"text":
            "Synthetic incident: At 10:00 UTC a configuration deployment completed. "
            "At 10:02 the API error rate increased. At 10:07 the team rolled back. "
            "Errors returned to normal at 10:09. The precise cause remains unconfirmed."}]}],
        "inferenceConfig": {"maxTokens": 180},
    }


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--invoke", action="store_true")
    args = parser.parse_args()
    request = request_body()
    Path("bedrock-request.json").write_text(json.dumps(request, indent=2), encoding="utf-8")
    if not args.invoke:
        print("Request written; no AWS call. Inspect bedrock-request.json.")
        return
    expected = os.environ.get("EXPECTED_AWS_ACCOUNT", "")
    if not re.fullmatch(r"\d{12}", expected):
        raise ValueError("Configure expected account")
    if "REPLACE" in request["modelId"] or request["modelId"] == "CONFIGURE_MODEL_ID":
        raise ValueError("Configure a supported model or inference profile")
    import boto3
    from botocore.config import Config
    from botocore.exceptions import BotoCoreError, ClientError
    runtime = None
    try:
        session = boto3.Session(region_name=os.environ["AWS_REGION"])
        if session.client("sts").get_caller_identity()["Account"] != expected:
            raise RuntimeError("Unexpected AWS account")
        runtime = session.client("bedrock-runtime", config=Config(
            connect_timeout=5, read_timeout=60,
            retries={"mode": "standard", "total_max_attempts": 2}))
        started = time.perf_counter()
        response = runtime.converse(**request)
        elapsed = time.perf_counter() - started
        content = response["output"]["message"]["content"]
        text = "\n".join(block["text"] for block in content if "text" in block)
        report = {"model": request["modelId"], "elapsed_seconds": elapsed,
                  "text": text, "stop_reason": response.get("stopReason"),
                  "usage": response.get("usage", {}),
                  "request_id": response["ResponseMetadata"].get("RequestId")}
        Path("bedrock-result.json").write_text(json.dumps(report, indent=2), encoding="utf-8")
        print(json.dumps(report, indent=2))
    except ClientError as exc:
        print(json.dumps({"aws_error": exc.response["Error"]["Code"],
                          "request_id": exc.response.get("ResponseMetadata", {}).get("RequestId")}))
        raise SystemExit(1)
    except BotoCoreError:
        print("SDK/credential/transport failure; inspect configuration securely")
        raise SystemExit(1)
    finally:
        if runtime is not None:
            runtime.close()


if __name__ == "__main__":
    main()
```

```powershell
python .\invoke_bedrock.py
Get-Content -LiteralPath .\bedrock-request.json
# Optional billable request after configuration and cost review:
python .\invoke_bedrock.py --invoke
if ($LASTEXITCODE -ne 0) { throw 'Inspect the reported failure before retrying.' }
```

## 4. Evaluate the response without assuming exact wording

Expected: a nonempty answer, a request identifier, a stop reason, and usage data for a compatible successful request. Read the text against a small rubric: two sentences; deployment precedes errors; rollback precedes recovery; no invented root cause; no incorrect timestamp. Exact prose and token counts vary by model and run.

If `stop_reason` indicates output truncation, the response may be incomplete even though the HTTP request succeeded. Increasing the output budget can help, but first check whether the task needs a shorter response. A refusal or filtered response is another application outcome to handle, not a JSON parser failure.

The client timeout is not proof that the service did no inference. Retrying an ambiguous failure can repeat inference and add cost. The SDK here permits a small bounded retry budget; it is not a per-user spending cap. Put request admission and per-user/application budgets around a production endpoint.

## 5. Run a controlled failure and extend carefully

For a local, free failure test, set `EXPECTED_AWS_ACCOUNT` back to the placeholder and run with `--invoke`. It must stop before creating a session or making an AWS request. Restore the actual value afterward. This verifies the guard, not IAM. An authorized negative IAM test requires a separate narrowly scoped role and an expected `AccessDeniedException` from the service.

For a streaming extension, use documented `converse_stream` with a model that supports it. Accumulate text deltas, handle error events, record first-token and completion latency separately, and preserve a cancellation path. Do not assume a partially displayed response is a complete successful result. For image/audio/video models, follow their supported content blocks and payload restrictions; a text-only request is not a universal multimodal API example.

For a model comparison, run the same fixed prompt set against two eligible candidates with the same rubric and explicit budgets. Record quality and task cost. Do not choose solely by one impressive answer or by maximum context length.

## Troubleshooting, cost, and teardown

`ValidationException`: check the ID type, supported inference mode, content format, and parameters. Access denial: inspect caller permissions, provider/model access, and destination restrictions. Throttling: reduce concurrency and inspect model/profile quotas. A missing `converse` method usually means the SDK is too old for the chosen API. A model returning tool blocks despite your intended task requires appropriate application handling; this example only accepts text for summarization.

Estimate input/output usage using the chosen model's current pricing. Profiles, special inference tiers, provisioned capacity, and optional logging have their own conditions. No persistent endpoint, knowledge base, or agent is created by this tutorial. Keep or remove the two local JSON files deliberately; they contain the synthetic prompt and response. If you enabled invocation logging separately, inspect its storage/retention rather than assuming the script removes it.

Explain: What does a successful invocation prove? Why configure a model ID externally? Why omit unnecessary sampling parameters? What could an inference profile change about data processing location? How would you handle a truncated response?

## Sources

- [Boto3 Converse API](https://docs.aws.amazon.com/boto3/latest/reference/services/bedrock-runtime/client/converse.html).
- [Converse inference](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html).
- [Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html).
- [Bedrock pricing](https://aws.amazon.com/bedrock/pricing/).

Reviewed 2026-10-18. Default execution is local; model availability and behavior must be checked when opting into AWS.
