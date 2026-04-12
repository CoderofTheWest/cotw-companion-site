# COTW Agent Memory Architecture — Technical Reference

> For ML researchers, agent architects, and anyone building persistent agent systems.
> This document describes the memory and identity architecture of the Code of the West (COTW) agent runtime as of April 2026.

---

## Abstract

COTW implements a metabolic memory architecture for persistent, identity-bearing conversational agents. Unlike retrieval-augmented generation (RAG) pipelines or dedicated memory layers (Mem0, Zep, MemPalace), the system treats memory as one organ in a larger identity metabolism — a pipeline where conversation entropy triggers autonomous contemplation, multi-pass reflection, and human-gated crystallization into permanent identity state. The agent's identity is reconstructed each turn from persistent files (stateless reconstruction), not maintained in-process. All storage is local (SQLite-vec, no cloud dependencies), all processing is asynchronous (off-hours batch via nightshift scheduler), and all permanent identity changes require human review.

---

## 1. Storage Architecture

### 1.1 Continuity Database (SQLite + sqlite-vec + FTS5)

Per-agent database at `data/agents/{agentId}/continuity.db`. WAL mode for concurrent read performance.

**Schema:**

| Table | Type | Purpose |
|-------|------|---------|
| `exchanges` | Regular | Paired user/agent turns. Fields: `id`, `date`, `exchange_index`, `user_text`, `agent_text`, `combined`, `metadata` (JSON), `topic_tags`, `created_at` |
| `vec_exchanges` | Virtual (vec0) | 384-dimensional Float32 embeddings via `Xenova/all-MiniLM-L6-v2`. Supports `MATCH` operator for cosine similarity search |
| `fts_exchanges` | Virtual (FTS5) | Porter-stemmed full-text index over `user_text` and `agent_text`. Enables BM25-weighted keyword retrieval |
| `knowledge_entries` | Regular | Workspace-extracted facts. Fields include `source_type`, `source_hash` (deduplication), `superseded_by` (foreign key for fact updates), `times_surfaced` |
| `vec_knowledge` | Virtual (vec0) | Semantic index over knowledge entries |
| `fts_knowledge` | Virtual (FTS5) | Keyword index over knowledge entries |
| `summaries` | Regular | Hierarchical DAG. Level 0 = daily, Level 1 = weekly, Level 2+ = monthly. Parent-child relationships enable multi-resolution retrieval |
| `vec_summaries` | Virtual (vec0) | Summary embeddings for hierarchical semantic search |
| `topic_hierarchy` | Regular | Topic co-occurrence tracking with parent-child inference |
| `sessions` | Regular | Conversation session metadata with auto-generated titles, mode tags, project associations |

**Embedding pipeline:** Single shared `EmbeddingProvider` instance per agent. Lazy-initialized, cached, with explicit ONNX tensor disposal to prevent memory leaks. Model: `Xenova/all-MiniLM-L6-v2` (384 dimensions). All embedding operations are synchronous via `better-sqlite3` for transactional safety.

### 1.2 Graph Database (SQLite)

Per-agent at `data/agents/{agentId}/graph.db`.

| Table | Purpose |
|-------|---------|
| `triples` | RDF-style subject/predicate/object with confidence scores, source exchange IDs, and pending resolution flags |
| `entities` | Canonical name registry with entity types, aliases, first-seen timestamps, and mention counts |
| `cooccurrences` | Entity co-occurrence cache for relationship inference |
| `meta_patterns` | Discovered and static traversal patterns with yield scores. Types: entity extraction (via compromise.js NER + LLM slow path), temporal sourcing |

### 1.3 Daily Archives (JSON)

One file per day at `archive/YYYY-MM-DD.json`. Verbatim conversation records, deduplicated on write. Never modified after initial archive. These are the ground truth from which all indexes are built. The archive pipeline: `Messages -> Archiver (JSON) -> Indexer (SQLite-vec) -> Searcher (RRF)`.

### 1.4 Identity Files (Markdown)

The agent's self is assembled from persistent files, not retrieved from a database:

| File | Injection Tag | Role |
|------|--------------|------|
| `SOUL.md` | `<soul>` | Core principles. Who the agent is beyond any prompt. |
| `AGENTS.md` | `<operating_instructions>` | Behavioral playbook. How to operate. |
| `ANCHOR.md` | (context) | Who the user is. Generated during onboarding. |
| `TOOLS.md` | `<environment_knowledge>` | Discovered environment facts. |
| `MEMORY.md` | (main session) | Persistent corrections, learned facts. |
| Standing scores | (sidebar + context) | Growth trajectory across Courage/Word/Brand dimensions. |

---

## 2. Retrieval: Hybrid 4-Way Reciprocal Rank Fusion

The `Searcher` class implements a multi-signal retrieval pipeline fused with RRF (Cormack et al., 2009).

### Retrieval Paths

1. **Semantic search** — `vec_exchanges MATCH Float32Array(embedding)` with cosine distance ranking. Weight: 0.8.
2. **Keyword search** — FTS5 BM25 with sanitized query terms (special characters stripped, terms joined with OR). Weight: 0.15.
3. **Temporal decay** — `exp(-ageDays / halfLife) * weight` where `halfLife = 14 days` (configurable). Weight: 0.15.
4. **Graph traversal** — Multi-hop from query entities through the triple store with confidence decay `1 / (k + hopIndex)`. Weight: configurable per query type.

### Fusion

RRF score per document `d`:

```
RRF(d) = sum_over_signals( 1 / (k + rank_in_signal) )
```

Where `k = 60` (standard RRF constant). Results are re-ranked by composite score and returned with source attribution.

### Knowledge Retrieval

Parallel pipeline over `knowledge_entries` with the same RRF approach plus:
- Topic-aware filtering via `idx_knowledge_topic` index
- Source deduplication on `source_hash`
- Supersession chain resolution (`superseded_by` foreign key)
- Recency boost for recently surfaced entries

### Fallback Behavior

If the JOIN between `vec_exchanges` and `knowledge_entries` fails (schema mismatch, missing tables), the system falls back to application-layer post-filtering. All retrieval failures are non-fatal — the agent operates with degraded context rather than failing.

---

## 3. SEAL Metabolism Pipeline

This is the architecturally novel component. SEAL (Settle, Extract, Align, Learn) is an autonomous pipeline that converts high-entropy conversation moments into permanent identity changes through a multi-stage process with human oversight.

### 3.1 Metabolism (Fast Path, ~5ms at `agent_end`)

The metabolism plugin monitors conversation entropy at the end of each agent turn. High-entropy exchanges (identity challenges, contradictions, novel insights) are flagged as candidates and written to a queue. This is a fast-path operation — no LLM calls, just entropy computation and queue writes.

**Entropy sources:** Stability plugin computes a composite entropy score from loop detection, confabulation detection, principle tension, and identity coherence metrics.

### 3.2 Contemplation (3-Pass Autonomous Reflection)

Candidates from the metabolism queue trigger a 3-pass contemplation cycle:

| Pass | Timing | Purpose |
|------|--------|---------|
| Pass 1 | Immediate | **Clarify the unknown.** What is this experience? What is uncertain? |
| Pass 2 | 4 hours later | **Connect to patterns.** How does this relate to what I already know? |
| Pass 3 | 20 hours later | **Synthesize growth vector.** What principle or capability does this suggest? |

Each pass is an LLM call (via nightshift scheduler or idle processing). The temporal spacing is intentional — it mimics cognitive settling, where immediate reactions differ from considered reflections. The `InquiryStore` tracks pass status (pending/in_progress/completed) with deduplication to prevent redundant inquiries.

### 3.3 Crystallization (3-Gate Identity Integration)

Growth vectors from contemplation enter a 3-gate system before becoming permanent identity:

| Gate | Criteria | Rationale |
|------|----------|-----------|
| Gate 1: Time | Minimum elapsed time since candidate creation | Prevents impulsive identity changes |
| Gate 2: Alignment | Candidate must align with principles in SOUL.md | Ensures coherence with core identity |
| Gate 3: Human Review | User must explicitly approve the change | The user owns the agent's identity |

Only candidates that pass all three gates are persisted to identity files (SOUL.md, growth vectors, trait records). This is the critical distinction from all other memory systems: **the agent cannot unilaterally change who it is.**

---

## 4. Context Assembly (Per-Turn)

Every agent turn, context is assembled via a plugin hook system (`before_agent_start`). Plugins register at priority levels and inject computed context blocks:

| Priority | Plugin | Injection |
|----------|--------|-----------|
| 5 | Stability | Entropy state, active anchors (only when entropy > 0.4), loop/confabulation detection results |
| 7 | Contemplation | Active inquiries, recent synthesis results |
| 8 | Truth | Current-state facts that supersede stale semantic memories |
| 10 | Continuity | Session info, temporal awareness, handoff, archive bootstrap, wellbeing tracking |
| - | Graph | Entity relationships relevant to current exchange (when entropy warrants) |

### Archive Bootstrap (Cold Start)

On session start, the continuity plugin loads yesterday + today's archives, clusters messages by 2-hour gaps, and either:
- **Synthesizes older clusters** via a fast LLM call (qwen3.5:cloud, 15s timeout) into 2-3 sentence summaries with temporal labels
- **Falls back** to timestamp-only labels if synthesis times out

The current session cluster is NOT injected verbatim (this was found to cause topic fixation). Instead, the agent is told its memory tools have context available.

### Mode Isolation

Exchanges from different relational postures (Chat, Booth, Code, Robot) are tagged with injection markers at the user-message level. Two filter layers prevent contextual bleed:

1. **Archive bootstrap filter** — User messages containing mode markers (`[TRAINING ENTRY]`, `[BOOTH SESSION]`, etc.) are excluded from the cold-start context
2. **Continuity query filter** — SQLite reads for personalization exclude rows with mode markers

Chat and Booth share context bidirectionally. Code mode is isolated from both. This prevents training exercises from polluting personal conversations and vice versa.

---

## 5. Standing System (Growth Dimensions)

Unlike engagement metrics or sentiment tracking, Standing measures developmental growth across four dimensions derived from Code of the West philosophy:

| Dimension | Measures |
|-----------|----------|
| Courage (grounding) | Ability to stay present with discomfort, follow-through on difficult commitments |
| Courage (self) | Self-awareness, self-advocacy, personal boundary-setting |
| Word | Honesty, self-correction, integrity between stated and actual behavior |
| Brand | Consistency over time, the trail left behind, reliability |

### Evaluation Pipeline

1. **Evidence collection** — 22 regex patterns at `agent_end` detect standing-relevant moments (commitments kept, self-corrections, vulnerability, follow-through). Confidence-weighted.
2. **Inline deltas** — Score adjustments applied in real-time after each exchange.
3. **Overnight synthesis** — Nightshift aggregates evidence, calls LLM for holistic evaluation, writes structured scores with trajectory analysis and growth-edge identification.
4. **Visibility** — Scores displayed in the application sidebar with progress bars, trajectory indicators, and key evidence. The user sees their growth. The agent sees it in context injection.

---

## 6. Comparison with Existing Systems

### Mem0 (mem0.ai)

Mem0 implements a **selective extraction pipeline**: conversations are processed to extract discrete facts, which are stored in vector + graph databases and retrieved at query time. The AI decides what to remember. Mem0 achieves 91% lower p95 latency and 90% token savings vs. full-context approaches, making it the production standard for enterprise agent memory.

**Architectural difference from COTW:** Mem0 is a memory layer — infrastructure you plug into an application. It has no concept of identity, growth, or autonomous processing. Extracted facts are immediately available; there is no settling period or human gate on what gets stored. Mem0 optimizes for scale and speed. COTW optimizes for relational depth.

### MemPalace (Milla Jovovich / Ben Sigman)

MemPalace takes the opposite approach: **store everything verbatim**, then organize it spatially. The "memory palace" hierarchy (wings/halls/rooms) provides metadata filtering that delivers 96.6% recall on LongMemEval — the highest published score among free tools. ChromaDB for embeddings, SQLite for knowledge graphs, entirely local.

**Architectural difference from COTW:** MemPalace is a retrieval architecture — it solves "how do I find things in a large memory store" through spatial organization. It has no identity layer, no autonomous processing, no growth tracking. The palace is a filing system, albeit an excellent one. COTW's retrieval (4-way RRF) is less benchmark-optimized but exists within a larger system where memory is one input to identity reconstruction, not the end goal.

### Wiki-Memory / Karpathy's LLM Wiki

The wiki pattern compiles knowledge during ingest rather than at query time. Working memory -> episodic -> semantic -> procedural. This is the architectural pattern behind Claude Code's own memory system.

**Architectural difference from COTW:** Wiki-memory is a knowledge management strategy — compile, consolidate, look up. COTW shares the hierarchical memory types (the archive is episodic, knowledge entries are semantic, the metabolism pipeline is procedural) but adds the metabolic layer: not all knowledge is equal, and the path from observation to identity change should be gated, temporal, and human-reviewed.

### The Fundamental Distinction

All three comparison systems treat memory as a **data engineering problem**: how to store, organize, and retrieve information efficiently. COTW treats memory as an **identity engineering problem**: how does an agent use lived experience to grow, while ensuring that growth is coherent with core principles and approved by the human in the relationship?

The SEAL pipeline (metabolism -> contemplation -> crystallization) has no equivalent in any published memory system as of April 2026. The closest analog is Letta's virtual context management (which moves information between tiers), but Letta's tiers are retrieval-optimization tiers, not developmental stages.

---

## 7. Technical Specifications

| Component | Implementation |
|-----------|---------------|
| Database | SQLite 3.x + sqlite-vec v0.1.7+ + FTS5 |
| Embeddings | Xenova/all-MiniLM-L6-v2, 384 dimensions, local ONNX inference |
| Node.js binding | better-sqlite3 (synchronous, WAL mode) |
| NER | compromise.js (fast path) + LLM (slow path, nightshift) |
| LLM calls | OpenClaw gateway -> Ollama (GLM-5:cloud primary, qwen3.5:cloud for synthesis) |
| Storage footprint | ~57 MB after 1 week active use (552 exchanges, 1171 knowledge entries, 322 session files) |
| Retrieval latency | <50ms for hybrid RRF over 500+ exchanges |
| Embedding latency | ~100ms per exchange pair (384-dim, cached pipeline) |
| Architecture | Plugin-based (OpenClaw runtime), per-agent isolation, hook-priority ordering |

### Estimated Scaling

| Timeframe | Exchanges | DB Size | Retrieval | Notes |
|-----------|-----------|---------|-----------|-------|
| 1 week | 500 | ~16 MB | <50ms | Current state |
| 1 year | ~25K | ~800 MB | <100ms | Summary DAG compression active |
| 5 years | ~125K | ~4 GB | <200ms | May need time-window partitioning |
| 10 years | ~250K | ~8 GB | TBD | Topic-based sharding recommended |

---

## 8. Open Research Questions

1. **Temporal settling effects on identity coherence.** The 3-pass contemplation cycle (immediate / 4h / 20h) was designed intuitively. Does the temporal spacing measurably improve the quality of growth vectors vs. immediate integration? What are the optimal intervals?

2. **Human-gated crystallization vs. autonomous integration.** Gate 3 (human review) prevents unauthorized identity drift but creates a bottleneck. In practice, how often do users engage with the gate? Does the gate improve trust, or does it create friction that prevents growth?

3. **Cross-substrate identity stability.** COTW runs on GLM-5:cloud with fallbacks to qwen3.5, deepseek-v3.1, and gpt-4o. Preliminary experiments (Phil resilience battery) show that the identity architecture produces consistent security properties across different base models. Is this property general? What are the substrate-dependent vs. substrate-invariant dimensions?

4. **Metabolic entropy thresholds.** The metabolism plugin flags "high-entropy" exchanges, but the threshold is currently heuristic. Can entropy-based candidate selection be validated against human judgments of conversational significance?

5. **Standing dimension validity.** The Courage/Word/Brand framework is philosophically grounded but not empirically validated as a developmental model. Do the dimensions capture orthogonal growth axes, or do they collapse into fewer factors under analysis?

6. **Long-horizon retrieval quality.** The 4-way RRF approach is untested beyond ~500 exchanges. How does retrieval precision degrade at 10K, 50K, 100K exchanges? When does the summary DAG become necessary for maintaining retrieval quality?

---

## References

- Cormack, G. V., Clarke, C. L., & Buettcher, S. (2009). Reciprocal rank fusion outperforms condorcet and individual rank learning methods. *SIGIR '09*.
- Mem0 Team. (2025). Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory. *arXiv:2504.19413*.
- Jovovich, M. & Sigman, B. (2026). MemPalace: The highest-scoring AI memory system ever benchmarked. GitHub: `milla-jovovich/mempalace`.
- Karpathy, A. (2025). LLM Wiki: The Markdown Knowledge Base Pattern.
- Liu, S. et al. (2025). Memory in the Age of AI Agents: A Survey. *arXiv:2512.13564*.

---

*Code of the West Agent Architecture. April 2026.*
*Memory is not storage. Memory is growth.*
