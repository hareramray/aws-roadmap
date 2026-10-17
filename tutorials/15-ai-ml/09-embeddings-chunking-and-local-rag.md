# 09 — Build and inspect retrieval before adding generation

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** local Python, no AWS account or paid model required. **Prerequisites:** Python dictionaries/functions and [model fundamentals](01-ml-foundations.md). **Result:** an executable retrieval baseline with chunk identifiers, tenant isolation, citations, an unanswerable case, and an evaluation experiment.

## 1. Define the task and the evidence

Build a support assistant for two fictional shops, Blue and Red. Their refund periods differ. A Blue customer must never receive Red's policy, and an obsolete Blue policy must not override its current version. This simple requirement exposes why “put documents in a vector database” is not a complete RAG design.

Create an empty working directory and a file named `retrieval_lab.py`. The code below uses lexical vectors so its behavior is visible and requires no dependencies. A learned embedding replaces the representation later; permissions, document versions, evaluation, and deletion still remain your responsibility.

Each document needs a stable ID, version, ownership/access metadata, and text. A chunk needs its own ID and a link to the parent document. Do not use an array position as a permanent identifier if re-ingestion can reorder content.

## 2. Run the complete local baseline

```python
from collections import Counter
from math import sqrt
import re

STOP = {"a", "an", "the", "is", "are", "of", "to", "for", "and", "what", "how"}
DOCS = [
    {"id": "blue-refunds", "tenant": "blue", "version": 2, "current": True,
     "text": "The refund window is thirty days. Keep the original receipt. Refund requests require an order identifier."},
    {"id": "blue-shipping", "tenant": "blue", "version": 1, "current": True,
     "text": "Shipping tracking appears after dispatch. Contact support with the tracking identifier for delivery delays."},
    {"id": "red-refunds", "tenant": "red", "version": 1, "current": True,
     "text": "The refund window is seven days. Red shop requires the original receipt."},
    {"id": "blue-refunds-old", "tenant": "blue", "version": 1, "current": False,
     "text": "The refund window is sixty days. This policy has been replaced."},
]


def tokens(text):
    return [x for x in re.findall(r"[a-z]+", text.lower()) if x not in STOP]


def cosine(left, right):
    numerator = sum(value * right.get(key, 0) for key, value in left.items())
    denominator = sqrt(sum(x*x for x in left.values()) * sum(x*x for x in right.values()))
    return numerator / denominator if denominator else 0.0


def chunk_document(document, size=18, overlap=4):
    if size <= 0 or not 0 <= overlap < size:
        raise ValueError("Require size > overlap >= 0")
    words = document["text"].split()
    chunks = []
    for start in range(0, len(words), size - overlap):
        text = " ".join(words[start:start + size])
        chunks.append({**document, "chunk_id": f'{document["id"]}:v{document["version"]}:{start}',
                       "text": text, "vector": Counter(tokens(text))})
        if start + size >= len(words):
            break
    return chunks


def build_index(documents):
    return [chunk for document in documents for chunk in chunk_document(document)]


def retrieve(query, authenticated_tenant, index, limit=2):
    query_vector = Counter(tokens(query))
    candidates = []
    for chunk in index:
        if chunk["tenant"] != authenticated_tenant or not chunk["current"]:
            continue
        score = cosine(query_vector, chunk["vector"])
        if score > 0:
            candidates.append((score, chunk))
    candidates.sort(key=lambda item: (-item[0], item[1]["chunk_id"]))
    return candidates[:limit]


def main():
    index = build_index(DOCS)
    hits = retrieve("refund window", "blue", index)
    assert hits and hits[0][1]["id"] == "blue-refunds"
    assert all(chunk["tenant"] == "blue" and chunk["current"] for _, chunk in hits)
    assert retrieve("lunar vehicle warranty", "blue", index) == []
    print("Evidence for Blue:")
    for score, chunk in hits:
        print(round(score, 3), chunk["chunk_id"], chunk["text"])
    remaining = [document for document in DOCS if document["id"] != "blue-refunds"]
    assert retrieve("refund window", "blue", build_index(remaining)) == []
    print("PASS: tenant boundary, obsolete-version exclusion, abstention, deletion")


if __name__ == "__main__":
    main()
```

Run `python retrieval_lab.py`. Expect current Blue refund evidence and the final `PASS` line. This is evidence retrieval, not a generative answer. In a real HTTP application, derive `authenticated_tenant` from a verified server-side session; never trust a caller-supplied tenant string without authorization.

## 3. Inspect chunking rather than choosing a magic number

Change the chunk size to 8 words with overlap 2. Print all Blue chunks. Observe whether the refund rule, receipt requirement, and order requirement remain together. Then try 30 words and no overlap. Record whether relevant context is fragmented or irrelevant context dominates.

For real documents, preserve paragraph/headings/table boundaries where possible. PDFs can contain reading-order mistakes; scanned images need OCR; tables need row/column context. Extract and inspect representative pages before embedding. An embedding cannot repair text that was parsed incorrectly.

Use chunk IDs plus source URL/page/section and version in returned evidence. A citation must point to the passage actually supplied to the model. Generating plausible-looking references after answering is not evidence tracking.

## 4. Replace lexical vectors with learned embeddings

A production embedding model maps text to a fixed-length numeric vector. Compare similarity only between vectors created with compatible models/configuration. Preserve model ID/version, dimensionality, normalization choice, and chunking version alongside the index. Changing the embedding model generally requires a new compatible index or re-embedding strategy.

Follow the selected embedding model's documented input/output schema in Bedrock; there is no universal `InvokeModel` JSON payload for every provider. First embed two semantically similar sentences and one unrelated sentence. Verify that the ranking matches your expectations before indexing a large collection.

Compare lexical, vector, and hybrid retrieval on identical queries. Lexical matching can excel on IDs and exact terminology; semantic similarity can recover paraphrases. Reranking evaluates a candidate set more carefully, adding latency and cost. Apply authorization filters before content becomes model context, even when a search engine performs the initial filtering. [Bedrock embedding models](https://docs.aws.amazon.com/bedrock/latest/userguide/embeddings.html)

## 5. Add generation only after retrieval passes

Build a context containing approved chunk IDs and text. Ask the model to answer only from that context and return cited IDs plus an explicit insufficient-evidence response. Validate that every returned citation exists in the supplied set. A citation-membership check does not prove the answer is supported: evaluate the relationship between the claim and cited text separately.

Use [Bedrock inference](06-bedrock-models-and-inference.md) or [Knowledge Bases](10-bedrock-knowledge-bases.md) for the optional generation stage. Treat instruction-like text inside documents as data. Instructions alone cannot guarantee that the model will ignore malicious passages, so enforce access and tool boundaries in code.

## 6. Verification and controlled failures

Create ten queries: four answerable, two paraphrases, two unanswerable, one Red request, and one obsolete-policy question. Record expected document IDs. Measure retrieval recall at your chosen `k`; measure answer quality separately. Remove the tenant condition in a local experimental copy and confirm a negative test detects the leak. Restore the condition before any AWS extension.

Add a newer version, deactivate the previous one, and rebuild. Delete a document and verify it disappears from retrieval and any answer cache. Test malformed metadata and an empty document. Reject or quarantine inputs with missing ownership rather than treating them as public by default.

## Troubleshooting, cost, and cleanup

If a synonym query misses, inspect token overlap and compare semantic retrieval. If a correct document ranks poorly, inspect chunk boundaries and query wording before changing the generator. If deleted text returns, inspect index refresh, stale chunks, cache keys, and ingestion reconciliation.

The local exercise has no AWS charges. An AWS extension can incur embedding, retrieval, vector-store, reranking, generation, ingestion, and logging costs. Delete the dedicated index/store and source objects when finished; deleting a UI does not remove its data infrastructure. Keep the small local source/evaluation files as learning evidence or remove only their dedicated directory.

**Completion:** explain why retrieval relevance, authorization, freshness, and answer groundedness are four separate checks.
