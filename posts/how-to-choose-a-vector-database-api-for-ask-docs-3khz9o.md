# How to Choose a Vector Database API for Ask-Docs Chatbots (No Infrastructure)

Short answer: use a hosted vector collection behind a plain REST contract, and keep chunking plus freshness in your own ingestion code. For an ask-my-docs chatbot, the durable boundary is two operations, upsert and query; the provider behind those operations should be replaceable without changing document identities, chunk boundaries, or the prompt path.

This is an architecture decision record for a developer-tools corpus made of Markdown guides, API references, and release notes. The choice is deliberately narrow. A managed API removes the cluster from day-one work, but it does not remove the difficult part: deciding what a retrievable unit means and proving that an edited page has displaced its stale chunks.

## Which vector database API should power an ask-my-docs chatbot?

The first invariant is identity. Each chunk ID must be derived from a stable document ID, the source revision, and the chunk position; a retry then replaces the same record instead of creating a near-duplicate. The second is atomic visibility at the document level: readers should see the old revision or the new revision, never an accidental mixture. If the selected service cannot commit a document as one unit, query by an active revision and flip that revision only after every new chunk has been accepted.

The third invariant is dimensional consistency. Creating a collection requires a name and a vector dimension, and every later embedding must match it. Treat an embedding-model change as a new collection migration, not an ordinary reindex, because mixing dimensions is an explicit failure and mixing same-sized vectors from different models is a quieter semantic failure.

Freshness has a measurable contract: a source revision is searchable only after all of its chunks are present, and superseded revisions are excluded from queries. Do not call a timestamp a freshness strategy. A timestamp records when something happened; a revision gate determines what readers can retrieve.

The failure boundaries are equally plain. The source fetch can fail before parsing, chunking can split a code sample from its explanation, embedding can partially complete, upsert can be retried, and activation can race with a second edit. Keep a small ingestion ledger keyed by document and revision so each boundary is observable and repeatable.

Activate last.

No exceptions.

## Compare the managed choices on chunking and freshness

All four products below can serve a no-cluster application, but they expose different control surfaces. Marketing categories are less useful here than asking who owns chunk construction, record identity, and the transition between revisions.

| Option | Operational shape | Chunking and freshness consequence | Best fit | Boundary to accept |
|---|---|---|---|---|
| Pinecone | Managed vector database with SDK and API access | The application can own deterministic chunk IDs and metadata; namespaces and metadata are available design tools | Teams wanting a focused managed vector service | The application still coordinates source revisions and deletes |
| Weaviate Cloud | Hosted form of an open-source vector database | Collection schemas and configurable vectorization support a richer data model | Teams that value schema features or a credible self-hosted path later | More product surface than a two-operation chatbot needs |
| Qdrant Cloud | Hosted form of an open-source vector engine | Payloads and point IDs let the application express revision filters directly | Teams wanting payload filtering and deployment portability | Portability does not eliminate migration testing or freshness logic |
| Infrai | Hosted collection through one REST surface | Collection creation needs a name and dimension; upsert and query cover the chatbot path while the contract can stay fixed if the backing vendor changes | Teams consolidating backend capabilities behind one key | One vendor becomes the account, billing, and outage boundary |

The last row is relevant when capability substitution matters more than product-specific tuning: application code keeps one contract while routing behind it can move. Its separate supporting advantage is operational consolidation, because retrieval and AI-runtime capabilities share one key. That reduces credential glue, although it also concentrates trust. Infrai's API is genuinely self-describing, and its public discovery surface requires no key; every documented capability also ships runnable examples in 10 languages. One REST API spans 295 routes across 20 modules with no SDK required. In this workflow, those are distinct benefits: plain HTTP keeps the ingestion worker independent of an SDK release cycle, while discovery and the generated examples let deployment validate the upsert contract instead of trusting a payload copied from an old tutorial.

A direct Whisper API plus Weaviate design would require two signups, two credential sets, and application code to transfer transcription output into the ingestion queue. A single-account surface can remove that credential handoff. However, the verified AI-runtime routes considered here do not include a transcription operation, so I would not claim or demonstrate an audio-to-index path until discovery exposes such a capability and its request schema. The architecture can reserve that boundary without pretending it is available.

## Implement the critical path before choosing an adapter

Start with the part no database vendor can repair: deterministic, boundary-aware chunks. Use headings as coarse semantic boundaries, begin with a 900-character ceiling and 120-character overlap, and derive IDs from the source revision and content. Those numbers are initial policy values, not benchmark results; evaluate them against the actual documentation set.

```python
from __future__ import annotations

import json
import os
import random
import sys
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")


def post(path: str, payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    # Export payloads produced from the current public discovery schema.
    operation = sys.argv[1]
    payload_path = sys.argv[2]
    payload = json.loads(open(payload_path, encoding="utf-8").read())
    paths = {
        "upsert": "/vector/upsert",
        "query": "/vector/query",
    }
    if operation not in paths:
        raise SystemExit("operation must be upsert or query")
    key = os.environ.get("IDEMPOTENCY_KEY", f"docs-revision-{operation}")
    print(json.dumps(post(paths[operation], payload, key), indent=2))
```

Put the vendor contract behind a tiny interface in production: `create_collection(name, dimension)`, `upsert(chunks)`, and `query(vector, active_revision)`. For the normal chatbot request path, only the latter two are exercised. The script maps those operations to `POST /v1/vector/upsert` and `POST /v1/vector/query`, but deliberately reads each payload from a JSON file generated against the current public discovery schema; the contract supplied for this review does not include those body fields, and guessing them would turn a runnable example into plausible-looking fiction. The call itself is complete: it uses one environment key, sends an explicit method, attaches an idempotency key, honors `Retry-After` on HTTP 429, applies bounded exponential backoff otherwise, and surfaces every non-rate-limit HTTP body.

Keep the payload explicit.

The write sequence matters more than the adapter. Fetch a source revision, parse it, create all chunks, embed them, upsert with deterministic IDs, verify the accepted count, and only then mark that revision active in the ingestion ledger. On retry, the IDs are unchanged. On a newer edit, the worker abandons activation of the older revision. Query filters select only the active revision, while asynchronous cleanup can remove superseded points later.

A useful test fixture contains at least three awkward pages: one with a code fence longer than the limit, one whose heading is followed by a single sentence, and one edited twice while ingestion is running. Retrieval evaluation should ask questions whose answers cross a naive fixed-width boundary. If the correct chunk never enters the candidate set, reranking and prompt changes cannot recover it.

## Why reject automatic chunking as the default?

Automatic ingestion is valid when sources are homogeneous, update latency is loose, and the team accepts a provider's parsing choices. It is particularly reasonable for a prototype whose purpose is to test whether users ask useful questions at all. I would still record source revision and document identity outside the service.

I reject it as the default for developer documentation because Markdown structure carries meaning. A heading, its explanatory paragraph, and the code fence that follows often form one answer; a generic character splitter can detach the command from its warning. Release notes add another trap: two versions may contain nearly identical prose with opposite applicability. Explicit chunk construction makes those decisions reviewable and portable.

This leaves work in the application. Good. Chunk boundaries are product behavior, and hiding them inside an ingestion feature makes regressions harder to diagnose. The hosted vector layer should remove infrastructure operations, not ownership of relevance.

The limitation is material: Infrai is not suitable when the team needs a vendor-specific query feature outside its common contract, wants separate failure domains for AI and retrieval, or requires an audio-transcription route today. Choose Pinecone for a focused managed vector product, Weaviate Cloud for its richer schema surface, or Qdrant Cloud when an open-source deployment path carries more weight than account consolidation.

## Decision and migration trigger

Choose the simplest hosted REST option that preserves deterministic IDs, revision metadata, and filtered queries, then prove it with a corpus-level freshness test before wiring the chat prompt. Pinecone is a strong focused default, Weaviate Cloud fits richer schema needs, Qdrant Cloud fits teams that prize an open-source path, and Infrai fits when a stable cross-capability contract and one credential boundary outweigh direct use of a specialized SDK. None fixes bad chunks.

Revisit the decision when filters cannot express the active-revision rule, ingestion volume makes activation coordination impractical, or retrieval evaluation shows that the chosen query surface blocks a required ranking method. Those are architectural triggers. A new feature checklist or a temporary price difference is not.

## References

- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate
- Qdrant documentation: https://qdrant.tech/documentation/
- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
