# Designing a Memory Infrastructure Layer

The previous chapters explored the problem space, design questions, and architectural patterns. This chapter describes one opinionated approach to building a memory infrastructure layer — a modular, open-source, Python-native module that separates storage concerns from intelligence concerns.

The principles behind this design:
- **Open-source and community-driven** — Apache 2.0, extensible by third parties
- **Python-native** — a library you import, not a service you deploy (though it can be deployed on K8s)
- **Framework-agnostic** — works with any agent framework, no coupling
- **Container-enabled** — runs in Docker, Kubernetes pods, or standalone
- **Extensible backends** — swap storage via configuration, add new backends without forking
- **Deterministic on the hot path** — no LLM calls during storage or retrieval

These are not the only valid choices. They represent one coherent set of trade-offs optimized for production deployments where explainability, portability, and operational simplicity matter.

## The API Surface

A memory infrastructure layer needs a small, composable API. The design goal: every common memory operation is one function call. No boilerplate, no framework setup, no configuration ceremony.

**Core operations:**
```
add(content, record_type, namespace, metadata)     → record_id
search(query, namespace, top_k, filters)           → ranked results
update(record_id, fields)                          → success
delete(record_id)                                  → success (soft delete)
```

**Composite retrieval:**
```
recall_context(query, namespace)
→ {similar_episodes, suggested_procedures, known_failures}
```

**Lifecycle and introspection:**
```
record_outcome(record_id, success)    → track effectiveness
list_namespaces()                     → what knowledge domains exist
stats(namespace)                      → counts, types, success rates
```

**Batch operations (for consolidation pipelines):**
```
get_by_session(session_id)            → all memories from a session
get_by_timerange(start, end)          → memories in a time window
get_stale(days, min_access)           → unused memories
bulk_archive(record_ids)              → soft-delete batch
```

This is intentionally minimal. Everything an agent needs to store, search, manage, and consolidate memory — without any extraction or intelligence logic baked in.

## Typed Records

Drawing from the cognitive science taxonomy discussed in earlier chapters, the storage layer defines typed records that carry structured fields alongside free-text content:

### Semantic Records — Facts and Knowledge
Stable domain knowledge. Decays slowly. High reuse value.
- Key fields: `confidence`, `source` (where the fact came from)

### Episodic Records — Events and Experiences
What happened, when, and what the outcome was. The most valuable type for pattern matching.
- Key fields: `event_time`, `duration_ms`, `outcome`, `actors`

### Procedural Records — Workflows and Runbooks
Reusable step-by-step knowledge. Can be human-authored (ingested from documentation) or agent-generated (learned from experience).
- Key fields: `steps`, `preconditions`, `tools_required`

### Why Types Matter

Typed records enable **composite retrieval** — searching for different kinds of knowledge in a single call. When an agent faces a new situation, it can ask: "What happened before? What procedure should I follow? What didn't work?" — three parallel searches by type, returned as a structured result. This is more useful than a flat list of mixed results.

Types also enable different lifecycle policies: episodic records might expire after 60 days without access, while procedural records persist indefinitely.

## Namespace-Based Scoping

Namespaces are hierarchical string paths that organize memories:

```
/sessions/{session-id}/     ← per-session (isolated by default)
/knowledge/{domain}/        ← curated shared knowledge
/users/{user-id}/           ← user preferences
/agents/{agent-id}/         ← agent-private learnings
```

**Implementation:** A text column with a prefix index. Search scoped to `/knowledge` matches all sub-namespaces via prefix matching. No namespace table, no configuration — namespaces are created dynamically when memories are written.

**Why hierarchical strings:** A tree data structure would require parent-child relationships, recursive queries, and schema overhead. String prefix matching achieves the same result with zero additional schema — just a column value and an indexed query. Simple, fast, portable across backends.

## Backend Abstraction

The storage layer defines an abstract interface that any backend must implement — a small set of methods covering storage, retrieval, lifecycle, and introspection. New backends register with the system and are selected by configuration:

```yaml
default_backend: pgvector    # change to "opensearch", "sqlite", etc.
```

The agent code never changes when backends are swapped. This is the core portability guarantee.

| Backend | Strength | Scale | Use Case |
|---------|----------|-------|----------|
| **PostgreSQL + pgvector** | HNSW cosine similarity, uses existing Postgres | Millions | Production, Kubernetes |
| **OpenSearch** | Hybrid vector + keyword search with rank fusion | Billions | When keyword matching matters alongside semantic |
| **SQLite + sqlite-vec** | Zero install, single file, portable | ~100K | On-device, development, air-gapped |

Adding a new backend (Redis, DynamoDB, Milvus, or any future vector store) means implementing the interface and registering it. No changes to the core library or existing backends. Third-party packages can add backends without forking.

## Policy-Based Scoring

Scoring is deterministic and configurable — no LLM in the retrieval path:

```yaml
scoring:
  weights:
    similarity: 0.5      # cosine similarity
    policy: 0.5           # usage signals
  policy_weights:
    success_rate: 0.4     # proven track record
    recency: 0.3          # freshness (power law decay)
    frequency: 0.3        # how often recalled
  decay:
    type: power_law       # or exponential
    alpha: 0.5
```

Every search result carries both the raw cosine score and the combined score. The agent or a layer above can apply additional reranking (including LLM-based reranking) on the returned results. The infrastructure provides a fast, explainable baseline; intelligence can refine it.

## Deployment Modes

The same library runs identically across deployment modes. The configuration determines the backend and embedding provider:

| Mode | Backend | Embeddings | Config |
|------|---------|-----------|--------|
| Development | SQLite (single file) | Local (Ollama) | `~/.config/config.yaml` |
| Docker | PostgreSQL container | Any cloud provider | `engram.yaml` |
| Kubernetes | Aurora or in-cluster pgvector | Cloud provider via IRSA/IAM | ConfigMap |

For Kubernetes deployments, the module ships as a Helm chart that handles:
- Database migration (enables pgvector extension, creates schema)
- ConfigMap with templated configuration
- Optional in-cluster PostgreSQL StatefulSet (for self-contained deployments)
- Support for both managed databases and in-cluster databases

The agent container installs the module (`pip install`) and reads configuration from the mounted ConfigMap. No separate server process — the module is a library imported by the agent.

## What This Layer Does NOT Do

Maintaining a clear boundary is as important as the features:

| Not this | Why | Who handles it |
|----------|-----|---------------|
| LLM-powered extraction | Intelligence layer, not infrastructure | The agent, or a pipeline above |
| Automatic contradiction detection | Requires content understanding | LLM pipeline, or A-Mem-style system |
| RL-trained memory policies | Requires training infrastructure | Agent framework, or AgeMem-style system |
| Full memory evolution (A-Mem style) | Requires similarity analysis + content updating | Processing pipeline above the storage |
| Knowledge graph construction | Requires entity extraction + resolution | Zep/Graphiti or similar |

The infrastructure layer surfaces similar memories (via search), supports supersession (via status fields), and exposes batch APIs for consolidation. But the decisions about what to extract, what contradicts what, and what to update are made by the layers above.

This boundary is not a limitation — it's a design choice that keeps the storage layer fast, deterministic, and portable. Intelligence layers change faster than infrastructure. By keeping them separate, you can swap your extraction pipeline without changing your storage, or swap your storage without changing your extraction.

## Key Takeaways

1. **Small API surface** — core operations (add, search, update, delete), composite retrieval, lifecycle tracking, and batch operations. No extraction logic.

2. **Typed records** enable structured retrieval. Episodes, procedures, and facts are searched separately and returned grouped.

3. **Namespaces are dynamic strings** with prefix matching. No configuration overhead. Session isolation by default.

4. **Backend abstraction** means swapping storage is a config change. New backends can be added by third parties without forking.

5. **Scoring is deterministic and configurable.** The formula combines cosine similarity with usage signals. No LLM on the retrieval path. Intelligence layers can rerank on top.

6. **The infrastructure-intelligence boundary is explicit.** The storage layer handles persistence, indexing, scoring, and lifecycle. Intelligence (extraction, evolution, RL policies) operates above it.
