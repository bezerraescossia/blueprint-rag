# Feature Specification: Document Ingestion Pipeline

**Feature Branch**: `E1F001-document-ingestion-pipeline`
**Created**: 2026-08-19
**Status**: Draft
**Epic**: .spec/E1-rag-system/epic.md — Feature F1
**Input**: User description: "F1 document ingestion pipeline"

## Business & Data Understanding

**Business Objective**: blueprint-rag is a generic, reusable RAG starter template — an adopter forks it and points it at their own document set. Today, doing that requires hand-building document loading, chunking, embedding, and index-population logic before any retrieval or Q&A is possible. This feature removes that cost: it gives any adopter a single, reproducible pipeline that turns a raw folder of documents into a queryable vector index.

**ML Objective**: Produce chunk-level dense vector representations of ingested documents, using an off-the-shelf embedding model, such that semantically relevant chunks are retrievable for arbitrary natural-language queries. This is a representation/indexing task, not a predictive-modeling task — there is no accuracy label to fit against here. Downstream retrieval quality (recall@k, MRR) is measured by F6 (Evaluation Harness) against the index this feature produces; this feature's own quality bar is indexing **correctness, completeness, and reproducibility**, not retrieval accuracy.

**Data Availability & Quality**: As a generic template, blueprint-rag has no fixed production document corpus — an adopter supplies their own at fork time. For building and testing this feature, a small representative sample corpus (a few dozen text/markdown documents) will be used as a development fixture; this is not real production data and carries no labeling requirement. The pipeline must not assume anything about document count, size, or subject matter beyond what's stated in Requirements below.

**Non-Goals**:
- Retrieval or query-time logic (embedding a query, ranking results) — that's F3 (Retrieval Service).
- Measuring retrieval quality (recall@k, MRR, faithfulness) — that's F6 (Evaluation Harness); this feature is scoped to indexing correctness and completeness only.
- OCR or scanned-image text extraction — documents must contain extractable text.
- PII detection, redaction, or access control — per the constitution's Data Governance section, documents are assumed non-sensitive by default.
- Fine-tuning or selecting between multiple embedding models at runtime — a single configured embedding model per deployment is assumed (model selection is a `sdd-plan` concern).

## User Scenarios & Testing

### User Story 1 - Ingest and Index a Document Set (Priority: P1)

An adopter points the pipeline at a folder of documents and runs it once, ending up with every document's content chunked, embedded, and stored in a vector index ready for retrieval.

**Why this priority**: Without this, there is no index and nothing downstream (retrieval, generation, evaluation) can function — this is the foundational walking-skeleton step of the entire epic.

**Independent Test**: Point the pipeline at a folder containing a known set of sample documents; after a single run, confirm every document produced at least one chunk, every chunk has a stored embedding, and the total chunk count in the index matches the ingestion run's own report.

**Acceptance Scenarios**:

1. **Given** a folder of 20 well-formed text/markdown documents, **When** the ingestion pipeline is run, **Then** the vector index contains chunks and embeddings for all 20 documents, and the run reports 20 documents processed with 0 failures.
2. **Given** an empty input folder, **When** the pipeline is run, **Then** it completes without error and reports 0 documents processed, rather than crashing.

---

### User Story 2 - Reproducible Re-Ingestion (Priority: P2)

An adopter re-runs ingestion after adding a few new documents to a folder that was already ingested, and the pipeline only processes what changed, without duplicating existing entries or losing reproducibility guarantees.

**Why this priority**: Constitution Principle II (Reproducibility by Default) requires versioned, deterministic data handling; without idempotent re-ingestion, every re-run risks silently corrupting the index with duplicate or stale entries, and DVC history becomes unreliable.

**Independent Test**: Ingest a folder, record the resulting chunk/embedding count, add 2 new documents and re-run ingestion, and confirm the index now contains exactly the original chunks plus the new documents' chunks — no duplicates of the unchanged documents.

**Acceptance Scenarios**:

1. **Given** a previously ingested, unchanged document folder, **When** ingestion is re-run, **Then** no new or duplicate chunks/embeddings are created, and the run reports the unchanged documents as skipped.
2. **Given** a document whose content has changed since the last ingestion, **When** ingestion is re-run, **Then** the document's stale chunks/embeddings are replaced with new ones derived from the updated content, and the run reports it as updated.
3. **Given** a completed ingestion run, **When** the resulting data and embeddings are inspected, **Then** they are versioned via DVC and the run's embedding-model identifier/version is recorded alongside them.

---

### User Story 3 - Handle Unsupported or Malformed Documents Gracefully (Priority: P3)

A folder being ingested contains a handful of files the pipeline can't parse (corrupted file, unsupported format, empty file); the pipeline skips just those files, logs why, and still successfully indexes everything else.

**Why this priority**: Real document folders are messy; failing the entire run over a handful of bad files would make the pipeline impractical for real use, but this is a robustness concern rather than a blocking one for the walking skeleton.

**Independent Test**: Ingest a folder mixing well-formed documents with a corrupted file and a file of an unsupported format; confirm the well-formed documents are indexed, the bad files are reported as skipped with a reason, and the process exits successfully.

**Acceptance Scenarios**:

1. **Given** a folder with 18 well-formed documents and 2 unsupported-format files, **When** ingestion is run, **Then** the 18 documents are indexed, the run reports 2 skipped files with reasons, and the process does not crash.
2. **Given** a document that parses but yields zero extractable text (e.g. a blank file), **When** ingestion is run, **Then** the document is recorded as skipped with an explicit "no extractable text" reason, rather than silently producing an empty chunk.

---

### Edge Cases

- What happens when a single document is far larger than the configured chunk size (e.g. a 500-page file)? The pipeline must still chunk and embed it completely rather than truncating it silently.
- How does the system handle a folder containing duplicate documents (identical content, different filenames)? Both should be tracked, but content-hash-based deduplication logic (or explicit indexing of both) must be a deliberate, documented behavior, not an accident.
- How does the system behave when the embedding model/API is temporarily unavailable mid-run? The run must fail loudly with a clear error and leave the index in a consistent state (no partially embedded documents silently left half-indexed), rather than continuing with placeholder data.
- How does the system behave when input data is stale relative to a previous DVC-tracked version? Re-ingestion must detect and version the change rather than silently overwriting history.

## Requirements

### Functional Requirements

- **FR-001**: The system MUST ingest documents from a configurable local directory, supporting at minimum plain text, Markdown, and PDF formats.
- **FR-002**: The system MUST split each ingested document into chunks according to a configurable chunk size and overlap policy.
- **FR-003**: The system MUST generate a dense vector embedding for every chunk, using a single configured embedding model per run.
- **FR-004**: The system MUST persist chunks and their embeddings in a vector index structured so that F3 (Retrieval Service) can query it directly.
- **FR-005**: The system MUST version the raw ingested document set and the resulting embeddings using DVC, per the constitution's Reproducibility principle.
- **FR-006**: The system MUST assign each document and chunk a stable, content-derived identifier so that re-ingestion of unchanged content is detectable and skippable.
- **FR-007**: The system MUST skip, rather than crash on, any document that fails to parse or yields no extractable text, and MUST record the skipped document's identifier and the reason.
- **FR-008**: The system MUST record ingestion run metadata — embedding model identifier/version, chunking parameters, timestamp, and per-run document counts (processed / skipped / updated) — for traceability and reproducibility.
- **FR-009**: The system MUST support true incremental re-ingestion: detecting new, modified, and removed documents against the last run, and re-processing only that changed subset rather than the full corpus.
- **FR-010**: Users MUST be able to inspect, after any run, which documents were processed, skipped, or updated, and why.

### Key Entities

- **Document**: see .spec/E1-rag-system/shared-data-model.md
- **Chunk**: see .spec/E1-rag-system/shared-data-model.md
- **Embedding**: see .spec/E1-rag-system/shared-data-model.md
- **Ingestion Run**: local to this feature — represents one execution of the pipeline. Captures the embedding model id/version used, chunking parameters, start/end timestamps, and per-document outcome (processed, skipped with reason, or updated). Referenced by FR-008 and FR-010; not shared with other features in the epic.

## Risk Assessment

| Failure Mode | Likelihood | Severity | Mitigation |
|---|---|---|---|
| Chunking strategy fragments semantically related content mid-thought, degrading downstream retrieval quality | Medium | Medium | Chunk size/overlap kept configurable; final defaults validated against F6's retrieval metrics before being treated as fixed, not assumed correct at design time |
| Embedding model version changes silently between runs, mixing incompatible vector spaces in the same index | Low | High | FR-008 mandates recording the embedding model id/version per run; re-embedding the full corpus is required on any model version change rather than mixing versions in one index |
| Data pipeline silently drops or fails to parse a subset of documents without visibility | Medium | High | FR-007 and FR-010 require explicit skip reporting with reasons; a run is not considered successful without a complete processed/skipped/updated accounting |
| Non-deterministic chunking or embedding calls break reproducibility (Constitution Principle II) | Low | Medium | Library/environment versions pinned; any stochastic parameters fixed; outputs DVC-versioned so any drift is detectable after the fact |
| Sensitive or unintentionally private content gets ingested despite the "non-sensitive documents" assumption | Low | Medium | Documented prominently as a constitution-level assumption (Data Governance); automatic PII detection is explicitly out of scope, so this risk is accepted, not silently mitigated |

## Success Criteria

### Business KPIs

- **SC-001**: A new adopter can go from a raw folder of documents to a queryable vector index using a single pipeline invocation, without writing any custom ingestion code.
- **SC-002**: Re-running ingestion against an unchanged document folder produces zero duplicate chunk or embedding records, verified across at least 3 consecutive re-runs.

### Model/ML Metrics

- **SC-003**: 100% of chunks produced from successfully parsed documents have a corresponding stored embedding, verified against the ingestion run's own chunk manifest, on the development sample corpus.
- **SC-004**: A reference corpus of 100 documents completes ingestion (parsing, chunking, embedding, indexing) in under 10 minutes on a single machine, with no manual intervention.

## Assumptions

- The development/test fixture corpus is a small (tens of documents), non-sensitive sample set curated for this feature's own testing — not a real production corpus, since none exists for a generic template.
- A single embedding model is configured per deployment; switching embedding models is treated as a full re-ingestion event, not a runtime choice (see Risk Assessment).
- Standard reproducibility defaults apply per the constitution: pinned environment/dependency versions, fixed seeds where randomness is involved, DVC-tracked data and embedding artifacts.
- The vector index technology, embedding model choice, and chunking algorithm specifics are deferred to `sdd-plan` — this spec deliberately does not choose them (WHAT/WHY, not HOW).
- Ingestion is a batch/offline process triggered by an operator or scheduled job, not a real-time/streaming pipeline reacting to individual file changes as they happen.
