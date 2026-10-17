# 16 — Compare document, speech, language, and image AI APIs

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** six small optional AWS experiments with synthetic inputs. **Result:** a comparison of a specialized API with a general foundation-model approach, including quality, latency, permissions, and cleanup. Check service/feature lifecycle and regional availability before each experiment.

## 1. Prepare one controlled fixture set

Create a fictional receipt containing a shop name, two items, quantities, prices, and a total. Render it as a clear supported image using your ordinary document tools. Record the known correct text and values. Also prepare one short spoken sentence, a short English support sentence, and a non-sensitive image you own.

Use no real identity documents, faces, personal conversations, or customer records. Set a per-experiment request bound, such as one baseline call and one controlled variation. Record exact filenames, expected content, language, and service configuration.

Use a named profile and explicit region in a Python session:

```python
import os
import boto3

session = boto3.Session(profile_name=os.environ["AWS_PROFILE"],
                        region_name=os.environ["AWS_DEFAULT_REGION"])
```

The code fragments below continue from this session setup and require the relevant action permissions. They are billable API instructions, not local simulations.

## 2. Translate: compare meaning, terminology, and format

Call Translate with the small synthetic support sentence:

```python
client = session.client("translate")
result = client.translate_text(
    Text="Your order has been dispatched. Keep the receipt for a refund.",
    SourceLanguageCode="en",
    TargetLanguageCode="hi",
)
print(result["TranslatedText"])
client.close()
```

Have a competent speaker review meaning, politeness, product terminology, and whether important conditions changed. Automated reverse translation can reveal issues but is not a substitute for human evaluation. Repeat with a brand term and investigate supported custom terminology when preservation matters. Inspect current language and input-size support rather than splitting text at arbitrary bytes. [Translate SDK guide](https://docs.aws.amazon.com/translate/latest/dg/get-started-sdk.html)

Expected evidence: a successful target-language response and a human quality note, not an assertion that every translation is exact. Clean up any custom terminology resources created for the extension.

## 3. Comprehend: inspect labels and confidence

For supported current features, call sentiment detection on two synthetic sentences:

```python
client = session.client("comprehend")
for text in ["The delivery arrived early and I am pleased.",
             "The product works, but the delayed delivery was frustrating."]:
    result = client.detect_sentiment(Text=text, LanguageCode="en")
    print(result["Sentiment"], result["SentimentScore"])
client.close()
```

Discuss why a mixed sentence can be harder than a clearly positive example. Confidence-like scores are not a guarantee that the label matches your business meaning. Investigate entity/key-phrase/language features only where currently supported. Do not infer a person's mental state or sensitive identity from an isolated sentiment label. [Comprehend developer guide](https://docs.aws.amazon.com/comprehend/latest/dg/what-is.html)

For classification, create your own labeled evaluation examples and measure errors by category. Remove optional custom models/endpoints or jobs separately if you create them.

## 4. Textract: recover document structure, not just words

Use your synthetic receipt image with a supported synchronous operation:

```python
from pathlib import Path

client = session.client("textract")
image_bytes = Path("synthetic-receipt.png").read_bytes()
result = client.detect_document_text(Document={"Bytes": image_bytes})
for block in result.get("Blocks", []):
    if block.get("BlockType") == "LINE":
        print(block.get("Text"), block.get("Confidence"))
client.close()
```

Compare line content with your known original. Rotate or degrade a copy and repeat once to observe OCR sensitivity. For tables/forms/queries or expense-specific extraction, choose the documented operation that exposes those structures; `DetectDocumentText` alone does not turn every receipt into validated accounting data. Respect supported synchronous formats/limits; larger multipage documents may require asynchronous S3-based jobs. [Textract guide](https://docs.aws.amazon.com/textract/latest/dg/what-is.html)

Validate extracted totals independently: two quantity/price pairs should reconcile with the stated total under your defined tax/rounding assumptions. Confidence does not replace business-rule validation. Remove input/output objects and optional job artifacts after the lab.

## 5. Polly and Transcribe: test a round trip

Choose a supported Polly voice/engine/language combination from the current voice list. Supply the voice ID through configuration and synthesize one sentence:

```python
from pathlib import Path

client = session.client("polly")
result = client.synthesize_speech(
    Text="Synthetic order one two three has been dispatched.",
    OutputFormat="mp3",
    VoiceId=os.environ["POLLY_VOICE_ID"],
    Engine=os.environ["POLLY_ENGINE"],
)
stream = result["AudioStream"]
try:
    Path("synthetic-speech.mp3").write_bytes(stream.read())
finally:
    stream.close()
    client.close()
```

Listen to pronunciation, number reading, pace, and accessibility. Compare supported SSML only after validating its markup and voice compatibility. Do not assume every voice works with every engine. [Polly guide](https://docs.aws.amazon.com/polly/latest/dg/what-is.html)

Upload that synthetic recording to a dedicated private S3 prefix. In Transcribe, start one batch job, specify its language/media format and supported input/output location, record the job ID, and poll with a bounded deadline until complete or failed. Inspect transcript words, punctuation, and timestamps against the known sentence. Calculate word error rate as `(substitutions + deletions + insertions) / reference words`, documenting your tokenization.

Do not reuse a Boto3 batch example as a streaming implementation: the streaming transport/SDK path differs. For a second optional experiment, use the supported console/streaming workflow and compare latency with batch completion. [Transcribe getting started](https://docs.aws.amazon.com/transcribe/latest/dg/getting-started.html), [SDK and streaming distinctions](https://docs.aws.amazon.com/transcribe/latest/dg/getting-started-sdk.html)

Delete the dedicated transcription job through the documented operation and separately delete input/output objects. Preserve only the small synthetic local recording if useful.

## 6. Rekognition: evaluate a narrow non-identifying task

Use a non-sensitive object/scene image you own. Call a supported label-detection feature and inspect labels, scores, and relevant bounding information. Compare against a manually labeled list of visible objects. Repeat with a crop or lighting variation, keeping a record of which image changed.

Treat returned labels as model outputs, not a complete inventory or proof about people. This exercise does not require face identification, emotion inference, or sensitive-trait classification. Inspect the lifecycle of the exact feature rather than assuming every feature of a long-lived service remains available. [Rekognition guide](https://docs.aws.amazon.com/rekognition/latest/dg/what-is.html)

## 7. Choose specialized API versus foundation model

Build a comparison table for the actual task: required output structure, supported languages/media, latency, batch/streaming, data handling, permissions, quality on fixtures, confidence/validation needs, price units, and lifecycle. A general model may offer flexible interpretation; a specialized API may provide a more focused structured contract. Neither wins every task automatically.

For a document assistant, a useful pipeline can be OCR → validated text/structure → chunking/retrieval → supported generation, with provenance preserved throughout. An extraction error upstream can become an unsupported answer downstream.

## Completion and cleanup

Deliver six short experiment records, the exact inputs/configuration, human or deterministic checks, observed failure cases, and a justified service choice. Remove only dedicated objects, custom resources, jobs, and logs. Count characters, audio duration, image/pages, API calls, optional custom endpoint time, and storage using current pricing. Keep calls bounded and avoid claiming a service is free because the sample is small.
