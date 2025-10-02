# RAG-LLM-System-Design
Below is a ready-to-save notes.md you can drop into your repo.

# RAG System Notes

Last updated: [today]

Overview
- Goal: A robust, cost-aware Retrieval-Augmented Generation (RAG) stack for technical manuals (Google Drive source).
- Stack: Unstructured parsing → structure-aware chunking → BGE/E5 embeddings → Qdrant/FAISS indexing → hybrid retrieval (dense + BM25 + RRF) → cross-encoder rerank → LLM with guardrails and citations.
- Priorities: Answer accuracy, faithfulness (citations), graceful fallback, and scalable costs.

Architecture (at a glance)
- Ingestion: Google Drive → loader → Unstructured (PDF/DOCX/OCR/tables) → normalization/dedup.
- Chunking: 800–1,200 tokens, 10–20% overlap; keep headings with first paragraph; tables as separate chunks with captions; payload includes source, page, section, element_type, chunk_id.
- Embeddings: BAAI/bge-large-en-v1.5 (or E5). Use prefixes: “passage: ” for docs, “query: ” for queries; normalize; cosine distance.
- Indexing: Qdrant (production, persistent) or FAISS (local prototype). Optional text index (BM25/FTS).
- Retrieval: Dense top-50; optional BM25 top-50; fuse via Reciprocal Rank Fusion (RRF); optional MMR; cross-encoder rerank to top 5–8.
- LLM: Free/local (e.g., Flan‑T5-small) or API. Strict “answer from context” prompt; must cite sources.
- Guardrails: sensitive query filter, score gating (cos ≥ 0.20), citation enforcement, ambiguity detection, fallback messages.
- Monitoring: Retrieval (P@k, R@k, MRR/NDCG, coverage), faithfulness (citation coverage, quote overlap), latency, safety blocks.

Mermaid architecture (for your slide)
```mermaid
flowchart LR
  subgraph ING[Ingestion & Preprocessing]
    A0[Google Drive / Files] --> A1[Loader]
    A1 --> A2[Unstructured Parser<br/>(PDF/DOCX, OCR, tables)]
    A2 --> A3[Preprocess & Clean<br/>(dedupe, normalize, filter)]
  end

  subgraph IDX[Chunking & Indexing]
    A3 --> B1[Structure-aware Chunker<br/>800-1200 tokens, 10-20% overlap]
    B1 --> B2[Embedding Model<br/>(BGE/E5 + &quot;passage:&quot;)]
    B1 --> B3[Payload: text, page, section, chunk_id]
    B2 --> B4[(Vector DB)<br/>Qdrant/FAISS (cosine)]
    B3 --> B4
    B1 --> B5[(BM25/FTS Index)<br/>(optional)]
  end

  subgraph RET[Query & Retrieval]
    C0[User Query] --> C1[Guarded Preprocess<br/>(normalize, PII check)]
    C1 --> C2[Query Embedding<br/>(BGE/E5 + &quot;query:&quot;)]
    C2 --> C3[Vector Search<br/>Top-50]
    C1 --> C4[BM25 Search<br/>Top-50 (optional)]
    C3 --> C5[RRF Fusion]
    C4 --> C5
    C5 --> C6[MMR Diversify (optional)]
    C6 --> C7[Cross-Encoder Reranker<br/>Top-5–8]
  end

  subgraph GEN[LLM & Guardrails]
    C7 --> D1[Context Builder<br/>(snippets + citation tags)]
    D1 --> D2[LLM Generation<br/>(Flan‑T5/local/HF)]
    D2 --> D3[Guardrails<br/>(citation enforcement,<br/>score gating, fallbacks)]
    D3 --> D4[Final Answer + Sources]
  end

  subgraph MON[Monitoring & Evaluation]
    C3 -.-> E1[Retrieval Metrics<br/>P@k, R@k, NDCG, coverage]
    C7 -.-> E1
    D3 -.-> E2[Faithfulness<br/>citation coverage,<br/>quote overlap]
    E1 --> E3[Logs / Dashboards]
    E2 --> E3
  end
```

Design trade-offs
- Parsing
  - Unstructured (hi_res) yields best table/figure fidelity; heavier dependencies and slower. Fallback to fast mode when OCR is not needed.
- Chunking
  - Larger chunks increase recall/context completeness but risk topical drift; smaller chunks improve precision but may fragment meaning. We target 800–1,200 tokens with 10–20% overlap and structure-aware rules.
- Embeddings
  - BGE-large gives strong recall/precision; heavier memory/latency. BGE-base/E5-base/MiniLM reduce cost at some accuracy loss. Use normalize + cosine; prefixes improve performance.
- Vector store
  - Qdrant: production-grade, hybrid-friendly, persistent. FAISS: great for local/offline prototyping, less operational flexibility.
- Retrieval method
  - Dense-only is simple/fast; hybrid (dense + BM25 + RRF) improves exact-term queries (error codes, part numbers) at added complexity.
- Reranking
  - Cross-encoder boosts precision; choose ms-marco MiniLM (fast) vs bge-reranker-base (more accurate, heavier). Use batching under load.
- LLM
  - Free/local (Flan‑T5) keeps costs low but outputs are shorter/less fluent; API LLMs improve fluency but add cost and governance considerations.
- Guardrails
  - Strict citation enforcement reduces hallucination risk but may increase “abstain” rate; thresholds tuned with validation set.

Retrieval strategy
- Chunking
  - Size: 800–1,200 tokens (1,100–1,700 chars). Overlap: 10–20%.
  - Keep headings with first paragraph; tables as separate chunks with captions; lists not split mid-item.
  - Payload: source, page, section_title, element_type, chunk_id.
  - Dedup repeated boilerplate (headers/footers/TOC).
- Embeddings
  - Model: BAAI/bge-large-en-v1.5 (or E5-large/base if preferred).
  - Index: “passage: ” prefix; Query: “query: ” prefix; normalize; cosine.
- Retrieval
  - Dense: top-50, score_threshold ≈ 0.20 (normalized).
  - Hybrid: BM25 top-50 fused with dense via RRF (k_rrf≈60) → optional MMR → cross-encoder rerank → final 5–8.
  - Rerankers: cross-encoder/ms-marco-MiniLM-L-6-v2 (fast) or BAAI/bge-reranker-base (higher quality).
- Faithfulness
  - Return source snippets and require citations [source: FILE, p=PAGE, id=CHUNK_ID].
  - Gating: if scores low or ambiguous, abstain and suggest clarifying question.

Guardrails & failure modes
- No relevant answers
  - Gate on scores: require any of {vector_score ≥ 0.20 OR rerank_score ≥ 0.0}. If not satisfied, return fallback: “Not found in the provided excerpts…”.
  - Ambiguity: if |top1 − top2| < 0.05 (rerank score), ask a clarifying question.
- Hallucinations
  - Strictly answer from snippets; enforce citations per sentence. If missing, defer with “Not found…”.
- Sensitive queries
  - Block unsafe intent keywords; lightweight PII patterns (emails/phones/credit cards). Return neutral refusal.
- Monitoring metrics
  - Retrieval: Precision@k, Recall@k, MRR/NDCG@k, coverage (any valid hit), deferral rate.
  - Faithfulness: citation coverage, quote overlap; optional NLI-support checks.
  - Safety: block rate by category; false block rate (manual review).
  - System health: P50/P95 latency (retrieve/rerank/LLM), error/timeout rates, score distributions.

Scalability considerations
- 10x documents
  - Dedupe and keep chunking tight; consider split-by-section/page first.
  - Qdrant: enable scalar quantization (INT8) or PQ; use HNSW m=24, ef_construct≈200; memmap large segments; compact regularly.
  - Incremental ingest: streaming upserts; background optimization; shard/replicate if needed.
- 100+ concurrent users
  - Separate ingest/index from query path; run Qdrant as a service.
  - Batch reranking; use MiniLM reranker for baseline, escalate only when needed.
  - Caching: query-result cache and answer cache; skip rerank for very high vector scores.
  - Tune hnsw_ef for latency/recall balance; use async I/O and micro-batching.
- Cost controls
  - Retrieval on CPU; switch to BGE-base/E5-base if needed.
  - Spot GPU for periodic embedding; serverless ingestion; progressive retrieval (dense-only → hybrid on-demand).
  - Tighten thresholds to control LLM usage; cache hot queries.

Prototype (minimal but runnable)
- Local-only stack for quick validation:
  - Vector store: FAISS (cosine on normalized MiniLM embeddings).
  - Embeddings: sentence-transformers/all-MiniLM-L6-v2.
  - LLM: google/flan-t5-small (Transformers pipeline).
  - Features: natural-language Q&A, snippets + citations, small P@k/R@k evaluation.
- See minimal prototype cell in the repo/notebook (FAISS + MiniLM + FLAN-T5).

Prompting (LLM)
- System-style instruction:
  - “Answer strictly from the provided context. Cite each sentence with [source: FILE, p=PAGE, id=CHUNK]. If info is insufficient, say ‘Not found in the provided excerpts.’”
- Add content constraints (no external knowledge) and style guidance (concise, bullet points, no speculation).

Default thresholds & knobs (tune on validation set)
- Retrieval: pre_k=50; final_k=5–8; vector score threshold≈0.20.
- Ambiguity delta: 0.05 (rerank score).
- Context: top_k_context=6; max_context_chars≈8,000.
- RRF: k_rrf≈60; MMR: lambda≈0.5 (optional).

Risks & mitigations
- OCR/table errors → fall back to fast mode; highlight low-confidence pages; manual review pipeline for critical docs.
- Drift in embedding quality → track score distributions and coverage; re-label/re-tune; consider hybrid when acronyms/IDs grow.
- Latency spikes → enable caching, micro-batching, and progressive retrieval; degrade gracefully (skip rerank).

Backlog / next steps
- Add lightweight query classification to route between dense-only vs hybrid vs domain-specific retrievers.
- Build small labeled set for offline evaluation; automate weekly P@k/R@k reports.
- Add quantization to embedding inference for cheaper CPU serving.
- Optional: integrate rewriter/rationalizer to improve answer readability while preserving citations.

Contact/ownership
- Components: Ingest (Owner A), Retrieval (Owner B), LLM & Guardrails (Owner C), Observability (Owner D).
- Rotation: Weekly sanity checks on coverage/deferral rates; monthly threshold tuning with labeled set.
