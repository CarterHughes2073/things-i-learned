# Vector Database API for Ask-My-Docs Chatbots: Python Freshness Without Infrastructure

TL;DR: For an ask-my-docs chatbot with no infrastructure team, choose a managed vector API by testing one narrow contract: upsert chunks with stable IDs and freshness metadata, filter during search, delete by document identity, and return enough metadata to cite the source. The least complex option is the service that passes that test from your Node.js application without a separate worker fleet or database tuning. In an edtech catalog, chunk quality and update propagation matter more than a long feature list.

This is a retrieval decision, not a database beauty contest. Retrieval-augmented generation separates external knowledge retrieval from generation, so weak retrieval gives the generator weak evidence. Start with a tiny evaluation corpus, instrument it, and keep the API boundary replaceable.

## Which vector database API should an ask-my-docs chatbot use?

A course-content chatbot needs four data operations: write a chunk, search eligible chunks, remove stale chunks, and fetch source metadata. Batch variants are useful, but they do not change the contract. Your application should own document identity, chunk identity, revision identity, and the text ultimately shown as evidence.

Start there.

The data flow is short. A publisher releases a lesson revision; an ingestion job normalizes its text, splits it along meaningful teaching boundaries, embeds each chunk, and upserts records tagged with the course, locale, publication state, and revision. At question time, the application embeds the query, applies access and freshness filters inside the search request, and sends only the selected text to the answer model. Citations come from stored source fields rather than model memory.

For a first notebook-to-production pass, I use a deliberately small adapter. The HTTP paths are deliberately absent; the important artifact is the application-owned request and response shape.

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    document_id: str
    revision: int
    course_id: str
    locale: str
    heading: str
    source_url: str
    text: str
    vector: list[float]


@dataclass(frozen=True)
class Hit:
    chunk_id: str
    score: float
    text: str
    heading: str
    source_url: str
    revision: int


class VectorIndex(Protocol):
    def upsert(self, chunks: list[Chunk]) -> None: ...

    def search(
        self,
        vector: list[float],
        *,
        course_id: str,
        locale: str,
        revision: int,
        limit: int,
    ) -> list[Hit]: ...

    def delete_document(self, document_id: str) -> None: ...


def answer_context(
    index: VectorIndex,
    query_vector: list[float],
    *,
    course_id: str,
    locale: str,
    live_revision: int,
) -> str:
    hits = index.search(
        query_vector,
        course_id=course_id,
        locale=locale,
        revision=live_revision,
        limit=6,
    )
    return "\n\n".join(
        f"[{hit.heading}]({hit.source_url})\n{hit.text}" for hit in hits
    )
```

This Python example is intentionally boring. Good. A Node.js caller can enforce the same typed boundary, while the evaluation harness can use this reference implementation to compare candidates with identical records and queries. The adapter prevents service-specific response fields from leaking into prompting, citation rendering, or authorization logic. I choose pre-filtering here because retrieving disallowed or obsolete chunks and removing them afterward wastes prompt space and creates an avoidable authorization boundary; the trade-off is that every candidate must support the fields and filter combinations the publishing model requires.

Do not infer deletion from an empty search result. Make it an explicit operation and test it. Likewise, do not make a course title the record key: titles change, locales collide, and a republished lesson can temporarily coexist with its predecessor. A stable document ID plus a monotonically increasing application revision makes those states visible.

## How should course material be chunked?

Chunk at instructional boundaries first: a worked example, a definition with its qualifiers, or a procedure with its prerequisites. Fixed token windows are a fallback for oversized sections, not the semantic plan. A chunk that starts halfway through a proof may be close to the query vector and still be useless to a learner.

Use measurements as experiment settings, not universal truths. An initial harness might compare 250-, 500-, and 800-token targets with a small overlap, then score whether the retrieved set contains the passage needed to answer each held-out question. The correct target depends on the material and embedding model. Record the splitter version alongside every chunk so a changed policy can trigger a controlled reindex rather than a mixed, invisible state.

I would keep parent context outside the embedded text when it adds noise, but preserve it as metadata and prepend a compact heading path when the lesson uses repeated labels such as "Example" or "Review." That is a real trade-off: more context can disambiguate a fragment, while repeated boilerplate consumes embedding and prompt space. The eval set decides.

Start with questions instructors already expect students to ask, plus adversarial cases: an answer present only in an unpublished lesson, two courses with similar terminology, an old revision that contradicts the live one, and a question whose answer spans adjacent chunks. For each query, store the expected document and acceptable chunk IDs. Track retrieval separately from answer quality; otherwise a fluent generator can hide a miss.

**The best chunking policy is the smallest one that clears the retrieval evaluation and preserves a readable citation.** It is not the policy with the most parameters.

## Freshness is a publishing transaction

A lesson update creates two risks: the new content may not be searchable yet, or old and new chunks may both be eligible. Treat publication as a state transition. Ingest the new revision under a revision value that is not yet live, verify that its expected chunks can be retrieved, then move the application's live-revision pointer. Queries filter on that pointer. Retire the prior revision after the switch succeeds. This ordering avoids asking the vector store to behave like the source-of-truth publishing database. The content system decides which revision is live; the search index serves records matching that decision. If ingestion is retried, deterministic chunk IDs such as `document_id:revision:splitter_version:ordinal` make the upsert idempotent. Keep the old revision for a bounded rollback window chosen by the team, not forever. The exact window is operational policy, so test both rollback and cleanup in staging. Picture the concrete failure: an instructor corrects a formula, the new chunks arrive, but an unfiltered query retrieves the old worked example because both revisions remain near the question in vector space. The answer can sound coherent while citing the superseded math. A live-revision filter prevents that ambiguity before generation. Freshness also needs an observable clock. Capture the content publication time, index completion time, and first successful retrieval time. Their differences expose where propagation slows down without pretending that one end-to-end number diagnoses the cause. Avoid promising an arbitrary number of seconds until the chosen service has been measured with your payload size, region, and concurrency.

Fresh means retrievable.

## A selection test beats a feature matrix

Create the same temporary corpus in every candidate service and run the same harness from the same application environment. Ten carefully labeled questions can expose basic contract failures; a production decision needs a corpus representative of real courses, locales, permissions, and update patterns. Do not use a provider's demo dataset because it cannot exercise your publishing mistakes.

The corpus wins.

Score four outcomes. First, can metadata filters exclude the wrong course, locale, publication state, and revision before results reach the prompt? Second, do repeated upserts and deletes converge on the expected records? Third, does the client expose timeouts and actionable errors so the application can retry writes without blindly retrying user queries? Fourth, can you export original IDs, vectors, text, and metadata in a form another adapter can ingest?

Then measure prompt cost. Log the number of retrieved chunks and the actual context size sent downstream, because `limit=6` does not imply six equally sized chunks. A larger retrieval set can raise the chance of finding evidence while also adding irrelevant text. Evaluate recall at several limits, then add reranking only if the measured errors justify another moving part.

Latency belongs in the harness too, but percentile measurements need enough samples and realistic concurrency. Report the distribution for your workload rather than a single attractive request. Separate embedding time, vector search time, and generation time; otherwise the vector API gets blamed for time spent elsewhere.

The final choice should be reversible. **Choose the candidate that satisfies the contract with the fewest application-side exceptions**, then pin its adapter behind integration tests. No vendor name can answer this in the abstract because filter behavior, limits, regions, and operational controls must be checked against current documentation and a live trial.

## Operate the boring path

Before launch, rehearse one complete publication, one corrected revision, one rollback, and one deletion request. Confirm that a search for each known question returns an allowed live chunk and that citation URLs resolve to content the learner may view. Inspect logs for document ID, revision, splitter version, filter set, returned chunk IDs, latency stages, and context size; exclude raw learner questions when retention policy does not permit them.

Set explicit timeouts at every network boundary. Retry idempotent ingestion with bounded backoff, send exhausted writes to a review queue, and make search failure visible to the answer layer so it can decline rather than fabricate an uncited answer. Alert on publication-to-retrieval lag and on evaluation regressions after changes to embeddings, splitters, filters, or prompts.

Finally, run the retrieval suite in continuous integration against the adapter's contract and on a schedule against a staging index. Keep a small fixed set for regression detection and a growing set built from reviewed failure classes. The notebook remains useful for exploration, but production earns its name when the same assertions run unattended.

For a no-infrastructure team, this is the practical endpoint: a managed API behind a thin adapter, application-owned freshness, eval-driven chunking, and citations assembled from retrieved records. Everything else must earn its place with a measured failure.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
