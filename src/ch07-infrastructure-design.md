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

| Backend Type | Examples | Strength | Scale | Use Case |
|-------------|----------|----------|-------|----------|
| **Vector database** | pgvector, Milvus | Cosine similarity with HNSW indexes | Millions | Semantic search over curated memories |
| **Search engine** | OpenSearch, Elasticsearch | Hybrid vector + keyword with rank fusion | Billions | When keyword matching matters alongside semantic |
| **Key-value store** | DynamoDB, Redis | Sub-millisecond exact lookups, TTL support | Unlimited | Session state, user defaults, fast retrieval by ID |
| **Embedded database** | SQLite + sqlite-vec | Zero install, single file, portable | ~100K | On-device, development, air-gapped |

Adding a new backend means implementing the interface and registering it. No changes to the core library or existing backends. Third-party packages can add backends without forking.

### Single Backend vs Multi-Backend Composition

The simplest deployment uses one backend for everything — vector search, session state, user defaults, and lifecycle management all in the same database. This is the right starting point. One connection to manage, one schema, one operational burden.

But as systems scale, **access patterns diverge:**

| Access Pattern | Characteristic | Best Served By |
|---------------|---------------|---------------|
| "What are this user's defaults?" | Exact key lookup, sub-millisecond | Key-value store |
| "Find memories similar to this incident" | Vector similarity, top-K | Vector database |
| "Show me all incidents containing 'connection pool'" | Full-text keyword search | Search engine |
| "Get all memories from session abc-123" | Range query on sort key | Key-value store |
| "What patterns exist across all users?" | Aggregation, analytics | Search engine |

When these patterns coexist, a single backend is either over-provisioned for some workloads or under-optimized for others. This is where multi-backend composition becomes valuable.

### The Composition Pattern

Rather than routing all operations through one backend, the composition pattern uses different backends for different access patterns, connected by a background processing layer:

```
Agent Interaction
    │
    ├── Fast writes ──→ Primary Store (key-value / relational)
    │                   Source of truth for all memories
    │                   Session state, user defaults, raw records
    │                   Sub-millisecond lookups by ID/session/user
    │
    │                        │
    │                        ▼ Background process
    │                   Extract insights, generate embeddings,
    │                   build semantic index
    │                        │
    │                        ▼
    └── Semantic search ──→ Search Index (vector / search engine)
                            Semantic similarity search
                            Hybrid keyword + vector queries
                            Analytics and aggregation
```

**The primary store** handles writes, exact lookups, session retrieval, and serves as the source of truth. It needs to be fast and reliable — key-value stores excel here.

**The search index** handles semantic retrieval, keyword search, and analytics. It's populated asynchronously from the primary store. It can be rebuilt from the primary store if needed — it's a derived view, not the source of truth.

**The background process** bridges them — extracting insights, generating embeddings, and indexing into the search layer. This is the same "sleep-time compute" pattern from Chapter 6, applied at the storage architecture level.

### When to Use Composition

| Start with... | Evolve to composition when... |
|---------------|-------------------------------|
| Single vector database (pgvector) | You need sub-millisecond session lookups AND semantic search, and one database can't optimize both |
| Single search engine (OpenSearch) | You need a cheaper, faster primary store for writes and exact lookups |
| Single key-value store | You need semantic similarity search that key-value can't provide |

**The decision framework:**

- **Predictable access patterns, moderate scale:** Single backend is sufficient. Don't add operational complexity you don't need.
- **Mixed access patterns, production scale:** Composition gives each workload the right storage engine. The operational cost of two systems is offset by better performance and cost efficiency.
- **Global, multi-region deployment:** Key-value stores with global replication serve as the primary store; regional search indexes handle local semantic queries.

### Composition in the Infrastructure Layer

The memory infrastructure module should support composition through configuration, not code changes:

```yaml
backends:
  primary:
    provider: dynamodb          # source of truth, fast lookups
    config:
      table_name: agent-memory
      region: us-west-2

  search:
    provider: opensearch        # semantic search index
    config:
      endpoint: https://search.internal
      index_name: memory-index

routing:
  writes: primary               # all writes go to primary
  exact_lookups: primary        # get by ID, session, user
  semantic_search: search       # similarity queries
  sync: background              # async replication primary → search
```

The agent calls the same API (`add`, `search`, `get`). The infrastructure routes operations to the appropriate backend based on the access pattern. This keeps the agent code unchanged whether running against a single SQLite file or a composed DynamoDB + OpenSearch architecture.

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

5. **Multi-backend composition** addresses divergent access patterns at scale. A fast key-value store for session lookups and exact-match retrieval, a vector/search engine for semantic similarity — connected by background processing. Start with a single backend; compose when access patterns diverge.

6. **Scoring is deterministic and configurable.** The formula combines cosine similarity with usage signals. No LLM on the retrieval path. Intelligence layers can rerank on top.

7. **The infrastructure-intelligence boundary is explicit.** The storage layer handles persistence, indexing, scoring, and lifecycle. Intelligence (extraction, evolution, RL policies) operates above it.
