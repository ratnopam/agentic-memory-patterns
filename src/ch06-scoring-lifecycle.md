# Scoring, Decay, and Lifecycle Management

A common surprise when first deploying vector search for agent memory: the results are semantically relevant but practically unhelpful. A search for "pod crashes" returns memories about pod crashes — good. But the top result is an untested observation from three months ago, while a proven fix from last week is buried at position four. Both have similar cosine scores; the difference is that one has been used successfully dozens of times and the other was never validated.

This reveals a fundamental limitation of pure vector search for agent memory. Cosine similarity captures text relevance but not trustworthiness, freshness, or track record. The scoring function needs to reflect whether a memory is worth acting on — not just whether it's semantically related to the query.

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
| **Hot** | Active session, recently used | Sub-millisecond | In-memory or fast key-value store |
| **Warm** | Recent sessions, frequently recalled | Low milliseconds | Primary database (vector or relational) |
| **Cold** | Historical, rarely accessed | Higher latency acceptable | Object storage, archival database |

Promotion (cold → warm on frequent access) and demotion (warm → cold after TTL) should be automated based on the same usage signals that drive scoring.

In practice, tiering often aligns with **multi-backend composition** — different storage engines for different tiers, connected by background processing. Hot memories in a fast key-value store for sub-millisecond session lookups, warm memories in a vector database for semantic search, cold memories in object storage for archival. Chapter 7 explores this composition pattern in detail.

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

## Closing Thoughts

## Closing Thoughts

Of all the improvements one can make to an agent memory system, the most impactful is often the simplest: access tracking. A counter that increments every time a memory is recalled, combined with power law recency decay, transforms retrieval quality more than sophisticated embedding models or prompt engineering.

Scoring doesn't need to be sophisticated to be effective. Start with cosine similarity plus recency plus access frequency. Add outcome tracking when agents begin reporting whether recalled memories were helpful. Consider learned scoring when the training infrastructure exists and the use case demands it.

Lifecycle management — the progression from active to deprecated to expired to archived — is the piece most teams skip. It's also the piece that matters most at scale. A memory system that only grows and never forgets will eventually drown in its own noise. The recommendation is to build lifecycle states in from the start, even if the initial policies are simple. Tuning policies later is straightforward; retrofitting lifecycle management into a system designed without it is significantly harder.
