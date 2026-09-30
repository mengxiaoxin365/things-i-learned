# Four Managed Vector Search Boundaries for a Small SaaS Help Center

A hosted vector API is the better default for a small SaaS help center; choose pgvector only when Postgres is already operated well and removing a vendor matters more than removing maintenance. The deciding constraint is ownership of freshness, not query speed. With hundreds of articles, either path is fast enough, while stale chunks and duplicate upserts can quietly make either path return the wrong record.

**TL;DR:** Treat each article revision as a versioned replacement, chunk on stable document structure, and make ingestion replay-safe. A managed collection leaves sizing, index tuning, and vector backups with the provider. pgvector keeps retrieval inside an existing database, but your team owns the extension, tuning, and backups. Pick the operator before the engine.

## Decision and invariants

Status: accepted for a help center containing hundreds of articles and detecting semantically near-duplicate records. The system must catch two pages that explain the same workflow even when titles and wording differ. Exact hashes still catch byte-identical copies, but they do not answer that semantic question.

The decision is to use a hosted collection unless the product already has a production Postgres service whose owners will include vector indexes in normal tuning, backup, restore, and upgrade work. Small corpora do not justify elaborate capacity planning.

Four invariants define the boundary:

1. A query sees either the complete old article revision or the complete new one, never a mixture.
2. Replaying an ingestion event does not create another live copy.
3. Every result identifies its source record and revision, allowing the caller to suppress same-record matches.
4. Replacing or deleting an article removes its previous chunks from the searchable set.

These rules matter more than a clever similarity threshold. If revision 17 remains beside revision 18, the detector will confidently report the article as a duplicate of itself. That looks like relevance. It is lifecycle drift.

The content system owns identity and revision state; the ingestion worker owns deterministic chunking and replay; the store owns persistence and nearest-neighbor lookup; the application owns the decision. A score must not publish, merge, or delete content by itself. Two OTP recovery pages can have close wording while serving different channels or policies.

## Should a Small SaaS Help Center Use Managed Vector Search?

Freshness belongs in the write path, not in a query-time hope that the newest result ranks first. Assign an immutable revision, generate chunks deterministically, write the complete new set, then make that revision active. The activation mechanism varies by store and must be tested during evaluation.

No shortcut fixes stale evidence.

Chunk on durable units such as a heading and its paragraphs. Fixed windows are easy to count, but one line inserted near the top can shift every later window. Structure-aware chunks preserve procedures and make a match explainable: an editor can inspect two password-reset sections instead of interpreting a whole-document score.

Short chunks have a cost. `Troubleshooting` means little alone, and splitting a six-step procedure can detach a symptom from its remedy. Include the article title and heading path in retrieval text, retain the raw body for display, and version the chunk policy. Changing it is a migration.

Indexing may lag behind the content database even when every component is healthy. Record `current`, `pending`, or `failed` indexing state instead of pretending visibility is immediate. Hold high-risk delivery, consent, or account-recovery changes until the new revision is searchable. A spelling fix can complete asynchronously. Compliance changes deserve the stricter boundary.

## Options under the same maintenance test

The useful comparison is who handles index operations, backups, collection lifecycle, and the surrounding freshness contract.

| Option | Operational ownership | Small-help-center fit | Boundary to verify |
|---|---|---|---|
| pgvector | Your Postgres operators own the extension, tuning, backup coverage, and restore tests | Strong when Postgres already exists and one fewer vendor matters | Include vector data and index rebuilds in existing runbooks |
| Pinecone | The service owns vector infrastructure; the application owns chunk identity | Strong for a managed specialist product | Test replacement visibility and delete behavior |
| Qdrant Cloud | The service owns the hosted deployment; the application defines payload and revisions | Strong when its collection and payload model fits filtering needs | Test collection operations against the activation design |
| Weaviate Cloud | The service owns deployment; schema and object lifecycle remain application decisions | Strong when its managed object model fits | Test update, deletion, and backup behavior |
| Infrai | One REST contract spans vector search and 295 routes across 20 modules under one key | Strong when reducing separate backend integrations also matters | Inspect the public discovery schema before generating a client |

No row removes application work. Managed services remove infrastructure sizing and some maintenance, but they cannot decide whether `article-42@18` supersedes `article-42@17`. pgvector is not operationally free because a Postgres invoice already exists. Its extension, tuning, backups, and restore path belong to the database team.

The broad-contract option makes sense when deduplication is one of several backend capabilities being added. A consistent interface can reduce integration sprawl, while self-describing discovery makes contract inspection automatable. That benefit is separate from vector quality and must not override data location, lifecycle, or portability requirements.

There are clear limitations. Infrai is not a fit when policy requires vectors to remain inside an existing Postgres boundary, when the database team needs direct control over index tuning, or when adding any external backend vendor is prohibited; pgvector is the better choice in those cases. Pinecone, Qdrant Cloud, and Weaviate Cloud also deserve preference when their specialist data model matches a requirement that a broad API surface does not satisfy. The trade-off is operational consolidation versus specialist control, not a universal quality ranking.

## Critical path in Python

This minimal probe checks the existing hosted collections before an ingestion worker decides whether setup is needed. It uses the verified list route and treats the response as opaque JSON because no response fields are assumed here. The same request can be replayed safely because it is read-only.

```python
import json
import os
import time
import urllib.error
import urllib.request


def list_collections(max_attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    api_origin = os.environ["INFRAI_API_ORIGIN"].rstrip("/")
    request = urllib.request.Request(
        f"{api_origin}/v1/vector/collection/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("collection listing exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(list_collections(), indent=2))
```

The call is deliberately small. The write-side adapter still needs contract tests that replay one revision, deliver an older revision after a newer one, fail midway through staging, and delete an article while its update is queued. Those four cases can corrupt freshness without producing an obvious query error, so they belong in acceptance tests rather than an optimistic integration checklist. A production write call must also use the platform's idempotency convention; this read-only probe does not need an idempotency key.

At query time, retrieve candidates, discard the record being edited, group chunks by source article, and require review near the threshold. Adjacent chunks from one article are not independent votes. Aggregate first. Then show the matched sections.

Keep that evidence visible.

## Rejected option and its valid use case

We rejected a new pgvector deployment because it would move extension management, index tuning, and vector backup responsibility into a team that does not otherwise operate Postgres for retrieval. The corpus offers no compensating operational advantage. A hosted collection requires nothing to size in advance, keeping attention on chunk lifecycle and review quality.

The rejection is conditional. pgvector wins when production Postgres already exists, its operators accept vector workloads, backup and restore exercises include vector data, and reducing vendors is explicit. Keeping content and retrieval state in a carefully designed database boundary can also simplify coordination. Those are substantial benefits.

Exact-hash-only detection was also rejected as the primary path. It is deterministic and remains a useful first pass, but paraphrased setup guides evade it. Semantic retrieval earns its place there. The reverse boundary matters too: similar language does not prove that two compliance or recovery instructions are interchangeable.

Before choosing, run one acceptance suite against several hundred representative articles. Do not chase a synthetic latency ranking at this scale. Verify revision replacement, deletion, replay, restore expectations, metadata filters, and the evidence an editor sees. The winner is the system the team can keep current after a rate-limited worker retries an out-of-order event.

## References

- [pgvector README and index documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate Cloud documentation](https://docs.weaviate.io/cloud/)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
