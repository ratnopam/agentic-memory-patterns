# Scoring, Decay, and Lifecycle Management

When an agent searches memory, it should see the most useful results first — not just the most semantically similar. This chapter examines how to rank memories, how they should age, and the full lifecycle management problem that remains largely unsolved across the industry.

## Beyond Cosine Similarity

Most vector search systems rank results by cosine similarity — how close the query embedding is to each memory's embedding. This captures semantic relevance but misses critical signals:

- **Is this memory trustworthy?** Has it led to good outcomes when used before?
- **Is this memory fresh?** Was it verified recently, or is it from months ago?
- **Is this memory proven?** Recalled 50 times, or never tested?

Two memories can match a query equally well semantically, but differ enormously in practical value. Scoring adds the dimensions that cosine similarity alone cannot capture.

## The Park et al. Scoring Function

Park et al. (2023) proposed the most cited scoring function for agent memory in their Generative Agents work:

```
Score = α × recency + β × importance + γ × relevance
```

Each component normalized to [0, 1]:
- **Recency:** Decay function based on time since last access
- **Importance:** A score assigned at write time reflecting the memory's significance
- **Relevance:** Cosine similarity between the query and the memory's embedding

This function established the principle that retrieval should consider multiple signals, not just text similarity. Most subsequent work has built on or responded to this foundation.

## Deterministic vs Learned Scoring

A fundamental design trade-off has emerged between two approaches to scoring:

### Deterministic Scoring

A formula with configurable weights:

```
score = W_similarity × cosine + W_policy × (
    W_success × success_rate +
    W_recency × recency_decay +
    W_frequency × normalized_access_count
)
```

| Property | Characteristic |
|----------|---------------|
| Latency | <10ms per evaluation |
| Cost | Negligible compute |
| Consistency | Perfectly reproducible |
| Explainability | "Score is 0.73 because: similarity 0.85, recency 0.65, success_rate 0.80" |
| Adaptability | Change weights via configuration |

### Learned Scoring (RL-Trained)

AgeMem (Yu et al., 2026) demonstrated that reinforcement learning can train memory scoring policies that outperform hand-crafted formulas on long-horizon benchmarks. The agent learns not just what to retrieve but also when to update, summarize, or discard memories.

| Property | Characteristic |
|----------|---------------|
| Performance | Empirically superior on benchmarks |
| Adaptability | Learns domain-specific policies |
| Consistency | Non-deterministic — same input may produce different rankings |
| Explainability | Opaque — "the model ranked it higher" |
| Requirements | RL training infrastructure, model-specific |

### The Complementary View

These approaches are not mutually exclusive. They operate at different layers:

- **Deterministic scoring is infrastructure.** It runs on every search, is fast and explainable, and works with any agent framework. It provides a reliable baseline.
- **Learned scoring is intelligence.** It can be layered on top — an RL-trained agent can use deterministic infrastructure underneath, adding its own reranking on top of the base results.

A practical architecture: the storage layer returns results ranked by deterministic scoring. The agent (or an RL-trained policy above it) can rerank the top-K results using learned preferences. The infrastructure handles the efficient narrowing; the intelligence handles the final selection.

## Decay Functions

The recency component of any scoring function requires a decay model. How fast should memories fade?

### Exponential Decay

```
recency = e^(-λ × days_since_access)
```

Drops sharply. A memory from 30 days ago with typical parameters is nearly invisible. Suitable for operational state that should expire quickly — current deployment versions, temporary workarounds, session-specific context.

### Power Law Decay

```
recency = max(1, days_since_access)^(-α)
```

Decays more slowly. A memory from 30 days ago retains meaningful signal. A memory from 6 months ago is still retrievable if the cosine similarity is strong enough. This matches the Ebbinghaus forgetting curve — the foundational model of human memory decay, which follows a power law, not exponential.

### Which to Choose

For most agent memory use cases, **power law is more appropriate.** An operations agent that resolved a complex incident six months ago should still be able to recall that experience when a similar incident occurs. Exponential decay would have effectively erased it.

Use exponential only for explicitly short-lived data — active deployment state, temporary configurations, session-specific context that should not persist.

The choice should be configurable, not hardcoded. Different namespaces or record types may warrant different decay characteristics — operational state decays fast, domain knowledge decays slowly, procedures may not decay at all.

## Lifecycle Management: The Unsolved Problem

Beyond scoring and decay, the full lifecycle question is the **largest unsolved problem** in agent memory. Every production system today either:
- Accumulates memories indefinitely (growing storage, degrading retrieval quality)
- Relies on manual cleanup (human operators periodically pruning)
- Uses fixed TTLs (hard expiration that loses valuable information)

None of these is satisfactory. A principled lifecycle management system should handle promotion, demotion, and archival automatically based on configurable policies.

### Memory Lifecycle States

```
                    ┌──────────────┐
                    │   ACTIVE     │
                    │              │
                    │ Full scoring │
                    │ In results   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
    ┌─────────────┐ ┌────────────┐ ┌──────────┐
    │ DEPRECATED  │ │  EXPIRED   │ │ ARCHIVED │
    │             │ │            │ │          │
    │ Superseded  │ │ TTL passed │ │ Cold     │
    │ by newer    │ │ without    │ │ storage  │
    │ memory      │ │ access     │ │          │
    │             │ │            │ │ Batch    │
    │ Hidden from │ │ Not in     │ │ access   │
    │ defaults    │ │ defaults   │ │ only     │
    └─────────────┘ └────────────┘ └──────────┘
```

### Soft TTL with Access Renewal

Hard expiration (delete after N days) loses valuable information. Soft TTL with access-based renewal preserves actively-used memories while letting unused ones fade:

```
Memory created → TTL starts (configurable per type)
    │
    ├── Accessed within TTL → TTL resets
    │   (memory stays active, proven value)
    │
    └── TTL expires without access → status moves to "expired"
        │
        └── After grace period → status moves to "archived"
            (cold storage, batch access only, never deleted)
```

TTL base values should vary by memory type:
- Episodic (events): moderate TTL, decays if not recalled
- Semantic (facts): longer TTL, stable knowledge
- Procedural (runbooks): very long TTL or indefinite
- User preferences: indefinite

### Signal Tracking

Lifecycle management requires usage signals tracked automatically on every memory:

| Signal | Updated When | Purpose |
|--------|-------------|---------|
| `access_count` | Every search that returns this memory | Frequency signal, TTL renewal |
| `last_accessed_at` | Every search that returns this memory | Recency signal, TTL renewal |
| `success_count` | Agent reports positive outcome | Success rate calculation |
| `failure_count` | Agent reports negative outcome | Success rate calculation |

These are updated by the storage infrastructure — no LLM calls, no intelligence decisions. Pure bookkeeping that enables both deterministic scoring and lifecycle transitions.

### Tiered Storage

As memory stores grow, not all memories need the same access characteristics:

| Tier | Contents | Access Pattern | Storage |
|------|----------|---------------|---------|
| **Hot** | Active session, recently used | Sub-millisecond | In-memory or fast database |
| **Warm** | Recent sessions, frequently recalled | Low milliseconds | Primary database (pgvector, etc.) |
| **Cold** | Historical, rarely accessed | Higher latency acceptable | Object storage, archival database |

Promotion (cold → warm on frequent access) and demotion (warm → cold after TTL) should be automated based on the same usage signals that drive scoring.

### The Never-Delete Principle

A consistent finding across the research: **memories should never be permanently destroyed.** Storage is inexpensive; lost information is irreplaceable. A memory that seems useless today may be the critical link for a future investigation.

Deletion should always be a soft operation — changing status to archived or expired, never physically removing the record. Even "wrong" memories have audit and diagnostic value.

## Consolidation: Processing Between Sessions

The lifecycle isn't just about decay and archival. It also includes **consolidation** — periodic processing that improves memory quality:

- Merging similar memories into richer records
- Generating summaries from groups of related episodic memories
- Updating importance scores based on accumulated usage patterns
- Identifying and resolving contradictions
- Promoting frequently-accessed cold memories to warmer tiers

This maps to what neuroscience calls hippocampal consolidation — the process that occurs during sleep, where the brain replays, strengthens, and reorganizes memories. In agent systems, this is "sleep-time compute" — processing that happens between sessions, not during them.

The storage layer should expose batch APIs that support consolidation workflows. The consolidation logic itself — what to merge, how to summarize, which contradictions to resolve — is intelligence-layer work, handled by the agent, an orchestrator, or a scheduled pipeline.

## Key Takeaways

1. **Cosine similarity alone is insufficient for ranking.** Add recency decay, success rate, and access frequency. The Park et al. formula is a strong starting point.

2. **Deterministic and learned scoring are complementary,** not competing. Infrastructure provides deterministic scoring on every search. Intelligence (RL, LLM) can rerank on top.

3. **Power law decay preserves long-term knowledge** better than exponential. Important experiences from months ago should remain retrievable.

4. **Lifecycle management is the biggest operational gap** in agent memory today. Implement lifecycle states, soft TTL with access renewal, and tiered storage.

5. **Track usage signals automatically** on every retrieval. Access count, recency, and outcomes are the foundation for both scoring and lifecycle management.

6. **Never permanently delete memories.** Soft-delete to archived. Storage is cheap; lost knowledge is expensive.

7. **Consolidation is a first-class concern.** The storage layer should support batch operations for between-session processing — merging, summarizing, promoting, and archiving.
