# Epic: RAG System

**Epic ID**: `E1-rag-system`
**Created**: 2026-08-19
**Status**: Draft
**Input**: User description: "rag system"

## Objective & Business Context

blueprint-rag exists to be a reusable, LangChain-based **starter template** for Retrieval-Augmented Generation systems — something an adopter can fork and point at their own document set rather than building a RAG pipeline from scratch. This epic covers the full walking skeleton of that template: getting documents in, indexing them, retrieving relevant context for a query, generating a grounded answer (or abstaining), serving that over an API, and proving it works via a mandatory evaluation gate. Success at the epic level means a fresh fork of this template can, with minimal adaptation, ingest an arbitrary non-sensitive document set and answer questions against it with measured, gated quality — not just a demo notebook.

## Scope & Non-Goals

**In Scope**: document ingestion and indexing, retrieval, grounded answer generation with abstention, a query-serving API, an evaluation harness satisfying the constitution's Evaluation Gate, observability/tracing built in from the start, and a basic feedback/monitoring loop.

**Non-Goals**:
- Any domain-specific document handling (e.g. construction blueprints, legal, medical) — this template stays generic per [[project_scope]].
- PII/regulatory compliance controls (access control, retention, redaction) — explicitly out of scope per the constitution's Data Governance section; deferred to whoever adapts the template.
- Multi-tenant or multi-user auth/authorization — a single-tenant query API is sufficient for the template.
- Fine-tuning or training a custom model — this epic assumes an off-the-shelf LLM and embedding model accessed via API/local inference, not model training.
- Choosing the experiment-tracking tool — still an open `TODO(EXPERIMENT_TRACKING_TOOL)` in the constitution; addressed as a flagged feature below, not assumed.

## Constitution Check

*Gate: must pass before the feature backlog below is considered final.*

| Principle (.spec/constitution.md) | Assessment | Note |
|---|---|---|
| I. LangChain as the Orchestration Framework, Used Plainly | Pass | Every pipeline feature (F1, F3, F4) is built on LangChain's own retriever/chain interfaces; no wrapper framework introduced. |
| II. Reproducibility by Default | Pass | F1 bakes in pinned environments, fixed seeds, and DVC-versioned data/embeddings as acceptance criteria, not an afterthought. |
| III. Mandatory Evaluation Gate | Pass | F6 exists specifically to satisfy this gate; F4/F5 are not considered done until F6 can measure them. |
| IV. Abstain Over Hallucinate | Pass | F4's goal explicitly includes abstention on low-confidence retrieval, not just generation. |
| V. Observability from Day One | Pass | F2 is sequenced immediately after F1, before retrieval/generation are built, so logging/tracing exists before there's anything interesting to log. |
| Data Governance | Pass | No PII/compliance feature is in this backlog by design — matches the constitution's explicit non-goal. |

## Feature Backlog

| ID | Feature Name | Goal | Priority | Depends On | Status |
|---|---|---|---|---|---|
| F1 | Document Ingestion & Indexing Pipeline | Load, chunk, embed, and store documents in a vector index, with DVC-versioned data/embeddings | P1 | — | Specified ([spec](../E1F001-document-ingestion-pipeline/spec.md)) |
| F2 | Observability & Tracing Foundation | Structured logging/tracing scaffolding (doc IDs, latency, token usage) that later features log into | P1 | — | Not Started |
| F3 | Retrieval Service | Embed a query and return ranked relevant chunks from the index | P1 | F1, F2 | Not Started |
| F4 | Grounded Answer Generation with Abstention | Synthesize an answer from retrieved context via LangChain, or explicitly abstain on low confidence | P1 | F3 | Not Started |
| F5 | Query Serving API | Expose the retrieve→generate pipeline as a callable API for external clients | P1 | F4 | Not Started |
| F6 | Evaluation Harness | Held-out eval set + retrieval metrics (recall@k, MRR) and generation metrics (faithfulness, answer relevance), enforced as the Evaluation Gate | P1 | F3, F4 | Not Started |
| F7 | Experiment Tracking Integration | Log runs/params/metrics from F6 (and future modeling work) to a chosen tracking tool | P2 | F6 | Not Started |
| F8 | Feedback & Monitoring Loop | Capture production query/answer signals and surface drift/quality alerts post-deployment | P2 | F5 | Not Started |

**Feasibility flag — F7**: blocked on `TODO(EXPERIMENT_TRACKING_TOOL)` in `.spec/constitution.md` — no tool is chosen yet. Resolve that TODO (constitution amendment, MINOR bump) before running `sdd-specify` on F7.

## Shared Entities

- **Document**: used by F1, F6
- **Chunk**: used by F1, F3, F6
- **Embedding**: used by F1, F3
- **Query**: used by F3, F4, F5, F6, F8
- **RetrievedChunk**: used by F3, F4, F6
- **Answer**: used by F4, F5, F6, F8
- **EvalExample**: used by F6, F7

See `.spec/E1-rag-system/shared-data-model.md`.

## Sequencing / Minimum Viable Epic

The minimum walking skeleton is **F1 → F2 → F3 → F4**: ingest and index a document set, have logging/tracing in place, retrieve relevant chunks for a query, and generate a grounded (or abstained) answer. That alone is testable end-to-end from the command line. F5 (Query Serving API) is the next slice that makes it consumable by an external client, and F6 (Evaluation Harness) is required before any of F3/F4 can be considered constitution-compliant enough to iterate on further. F7 and F8 are explicitly deferred past the initial walking skeleton.

## Assumptions

- An off-the-shelf embedding model and LLM (local or API-based) are available to the adopter; this epic does not cover provisioning or fine-tuning either.
- The initial document corpus used to build and evaluate the pipeline is small enough to iterate on locally (no distributed indexing infrastructure assumed).
- "Vector index" is treated generically here — the specific vector store product is a decision for F1's `sdd-plan`, not this epic.
- Documents are assumed non-sensitive, consistent with the constitution's Data Governance section.
