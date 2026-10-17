# 10 — Build a versioned, access-aware Bedrock knowledge base

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** optional billable AWS exercise after the [local retrieval lab](09-embeddings-chunking-and-local-rag.md). **Result:** an ingested synthetic document set, inspected retrieval results, an evaluated answer, and tested update/deletion behavior. Keep the collection to a few short text files.

## 1. Choose a concrete deployment path

Knowledge Bases offers different setup and ownership paths. Select the current workflow available in your region/account and record who owns storage, indexes, parsing, ingestion, and IAM. This tutorial's data layout and SDK retrieval example target a supported vector knowledge base with an S3 data source and metadata filtering. Do not assume every connector, managed mode, or store exposes identical capabilities. [Knowledge Bases guide](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)

Before creation, list the exact resources that the wizard will create, the minimum ongoing charges, deletion behavior, and whether provisioned search/database capacity remains billable while idle. Do not accept a default store without understanding these costs. For a restricted budget, complete the local tutorial and this design/permission exercise without provisioning.

## 2. Prepare three documents and their metadata

Create `blue-refunds.txt`, `red-refunds.txt`, and `blue-shipping.txt` with the synthetic policies from tutorial 09. Use no personal or business data. Put them under a dedicated private S3 prefix such as `ai-lab/source/` using the established private-bucket workflow.

For a supported S3 metadata sidecar configuration, place `blue-refunds.txt.metadata.json` beside `blue-refunds.txt`:

```json
{
  "metadataAttributes": {
    "tenant": "blue",
    "document_id": "blue-refunds",
    "policy_version": 2,
    "active": true
  }
}
```

Create matching metadata for the other documents. Check the current supported metadata types/size and reserved keys for your chosen path. A typo in a sidecar filename can produce missing metadata rather than the authorization behavior you expected. Inspect the indexed result instead of trusting the source filename. [S3 data-source setup](https://docs.aws.amazon.com/bedrock/latest/userguide/s3-data-source-connector.html)

## 3. Separate the three identities

Your **provisioning identity** creates/configures the knowledge base and passes its approved service role. The **knowledge-base service role** reads only the source prefix, invokes the chosen embedding model, and accesses the selected store and relevant keys. The **application runtime identity** retrieves from the intended knowledge base and, if needed, invokes an approved generation model.

Draw these as separate arrows. Constrain trust to the appropriate service/account/resource where supported. If the bucket, key, and index use different policies, verify every layer. `iam:PassRole` does not itself allow a service role to read S3. Conversely, an S3 allow cannot bypass a KMS deny. Use the exact current service-role guide for the selected backend. [Knowledge Base service-role permissions](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-permissions.html)

Keep the application runtime credentials on the trusted server. End users must not receive those credentials or direct permission to retrieve from the shared knowledge base without the application's authorization checks. A filter supplied by application code cannot protect a caller who can bypass that code and make an unfiltered request.

## 4. Create and ingest

1. Open Bedrock in the chosen region and choose the supported Knowledge Bases creation flow.
2. Set a unique lab name and the approved service role; record the knowledge-base ID.
3. Select the embedding model supported by the chosen store and region. Record its ID and output dimensionality.
4. Configure the S3 data source for only the dedicated prefix; choose and document chunking/parsing settings.
5. Inspect storage/index configuration, metadata filtering support, network access, and encryption before creation.
6. Start ingestion/synchronization. Record job ID, status, failure reason if any, and counts of processed/failed documents.
7. Do not query until ingestion succeeds. Verify a known sentence through retrieval and inspect source location plus metadata.

An ingestion success count is necessary but insufficient. It does not prove correct parsing, ownership metadata, or semantic retrieval quality.

## 5. Query with a server-derived filter

Install `boto3` in the tutorial virtual environment and supply the known ID, profile, and region through environment variables. Save as `retrieve_kb.py`. The fixed tenant below represents an already-authorized test identity; a real application must derive it from validated authentication, not a query parameter.

```python
import os
import boto3

session = boto3.Session(profile_name=os.environ["AWS_PROFILE"],
                        region_name=os.environ["AWS_DEFAULT_REGION"])
client = session.client("bedrock-agent-runtime")
tenant = "blue"
response = client.retrieve(
    knowledgeBaseId=os.environ["KNOWLEDGE_BASE_ID"],
    retrievalQuery={"text": "What is the refund window?"},
    retrievalConfiguration={"vectorSearchConfiguration": {
        "numberOfResults": 3,
        "filter": {"andAll": [
            {"equals": {"key": "tenant", "value": tenant}},
            {"equals": {"key": "active", "value": True}}
        ]}
    }}
)
results = response.get("retrievalResults", [])
for result in results:
    metadata = result.get("metadata", {})
    if metadata.get("tenant") != tenant or metadata.get("active") is not True:
        raise RuntimeError("Retrieval authorization metadata mismatch")
    print(metadata.get("document_id"), result.get("score"))
    print(result.get("content", {}).get("text", ""))
if not results:
    print("Insufficient authorized evidence; do not invent an answer")
client.close()
```

Expect Blue's current policy and no Red content. Check whether your selected backend supports these filter types before running. A similarity score is not a calibrated probability. For a production endpoint, retain the second authorization check and fail closed on missing metadata. Resource-level authorization might require stronger isolation than a shared index, depending on the threat model. [Retrieve API](https://docs.aws.amazon.com/boto3/latest/reference/services/bedrock-agent-runtime/client/retrieve.html)

## 6. Generate and inspect citations

For the defense-in-depth application path, call **Retrieve → validate every result's authorization metadata → generate using only those validated results**. Stop before generation if any result fails validation; abstain when no authorized evidence remains. Choose an available generation model and record its identifier/configuration.

If using `RetrieveAndGenerate`, explicitly supply the same server-derived `tenant` AND `active` filter in that request's `retrieveAndGenerateConfiguration.knowledgeBaseConfiguration.retrievalConfiguration.vectorSearchConfiguration.filter`. It does not inherit the filter from an earlier `Retrieve` call. This combined API does not give your application a chance to validate each retrieved chunk before generation, so use the separate-call path when that extra check is required. Inspecting citations after generation cannot undo disclosure. [RetrieveAndGenerate API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html)

For a console test, configure and verify equivalent filters on its generation retrieval. If the console path cannot demonstrate those filters, test generation only against a separate single-tenant synthetic collection. Do not use an unfiltered console test against the mixed Blue/Red collection.

Ask the same question, then an unanswerable question. Inspect the returned source references and check that the refund period is supported by a current authorized passage. Keep conversational session state tied to the same principal/tenant; do not reuse another user's session ID. Limit output length and inference work.

## 7. Prove changes and negative cases

Change Blue's policy to a new version and sync. Verify the expected version is retrieved. Remove an input document and perform the selected data-source deletion/sync process; verify retrieval absence and invalidate any cached answers. Explicitly test a user losing access between conversations.

Introduce one malformed sidecar and observe ingestion/error behavior. Restore it and re-ingest. Ask a question whose wording strongly matches Red's text while signed in as Blue; the filter must still exclude Red. Add a harmless prompt-injection sentence to a test document and confirm it cannot authorize tool execution.

## Troubleshooting and cleanup

For ingestion denial, inspect the service role, source prefix, key policy, model access, and index permissions. For missing evidence, inspect parsing, chunking, metadata type, filter support, sync status, and source version. For generation denial, inspect the application identity/model permission separately from ingestion.

Delete the dedicated knowledge base/data source through its documented workflow and separately inspect source objects, vector indexes/collections/databases, logs, secrets, and IAM roles created for the lab. Record what the wizard retained. Do not remove shared stores, keys, or roles. Recheck costs after billing data arrives.

**Completion:** show the same question under two authorized tenants, a document update, a deletion, an unanswerable request, and a full resource ledger.
