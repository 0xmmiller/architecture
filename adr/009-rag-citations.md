# ADR-009: RAG over owned documents, with citations

- Status: Accepted
- Date: 2026-09-18
- Deciders: Mark Miller

## Problem

Atlas agents that only call `get_portfolio` cannot answer "why did we
pick Kafka?" without stuffing ADRs into the prompt. Naive "chat with
PDF" dumps chunks without provenance and silently mixes chain facts
with docs.

## Options considered

### 1. Stuff the system prompt

Works until the doc set exceeds the window. No citations. No eval.

### 2. Hosted vector DB (Pinecone-class)

Fine at scale. Wrong default for a lab that already runs local models
(ADR-008). Another bill, another failure domain.

### 3. Local embeddings + in-process (then pgvector) store

Chunk markdown, embed via Ollama `/api/embeddings` (or a hash embedder
in CI), retrieve top-k with cosine, **return source paths**. AgentKit
exposes `retrieve` as a tool. Atlas docs are the first corpus; chain
state stays on the portfolio tool, not in the vector index.

## Decision

**RAG is a retrieval tool with citations, not a chatbot.**

- Corpus: owned markdown (ADRs, runbooks), not the live chain.
- CI: deterministic hash embeddings. Laptop: Ollama embed model.
- Every hit carries `path` + `score`. The model must quote paths.
- Eval: a golden question set in `ragkit` with recall@k.

## Trade-offs

- **Accepted:** two memories (docs vs chain). Mixing them is a bug.
- **Rejected:** embedding the entire Kafka log.
