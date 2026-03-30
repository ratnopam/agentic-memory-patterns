# Memory Formation and Evolution

This chapter addresses two of the most consequential design decisions: when should an agent store to memory, and how should stored knowledge change over time.

## The Trigger Problem

A common early mistake when building agent memory: storing a record after every successful tool call — every kubectl output, every log snippet, every metric check. Within a day, the memory store has thousands of records. When the agent searches for "OOM kills," it gets back a wall of raw output fragments instead of the one concise record that actually captures the root cause and fix.

The problem isn't storing too much — Park et al. (2023) proved that storing everything can work. The problem is storing the wrong *kind* of everything. Raw tool outputs aren't memories. A memory is the distilled knowledge: "OOM kills in payment-service caused by connection pool leak. Fix: set maxPoolSize=50." That distinction between raw data and curated knowledge turns out to be one of the most important design decisions in any memory system.

What should an agent store in long-term memory? The answer depends on the architecture's philosophy toward memory formation.

## Five Trigger Patterns

### Store Everything, Filter at Retrieval

Park et al. (2023) took the most aggressive approach in their Generative Agents work: every observation becomes a memory record. Mundane events and significant events alike are stored. The sophistication is entirely in the retrieval scoring function — recency, relevance, and importance determine what surfaces when memories are searched.

This eliminates the "forgot to store" failure mode. Importance is assessed retrospectively, not predicted. The trade-off is that storage grows with every observation, and retrieval quality must be excellent to surface the right memories from a large store.

### Agent-Initiated Storage

MemGPT (Packer et al., 2023) gave agents explicit memory tools and let the agent decide, during its reasoning, what to save. The agent manages its own memory like an operating system manages virtual memory — deciding what to keep in active context and what to archive.

This is elegant but introduces non-determinism. The same situation may or may not trigger storage depending on the LLM's reasoning in that moment. Important information occasionally goes unsaved.

### Event-Driven Storage

Production systems often trigger storage on system events rather than agent decisions: task completed successfully, error encountered, user provided explicit feedback, session ended. This is deterministic — events always trigger storage. The "what to store" decision may still involve an LLM (to summarize or extract), but the "when" is handled by the orchestrator.

### RL-Trained Policies

AgeMem (Yu et al., 2026) demonstrated a significant advance: using reinforcement learning to train memory management policies. The agent learns WHEN to store, update, and discard through a three-stage progressive training strategy. Six memory operations (add, update, delete for long-term; retrieve, summarize, filter for short-term) are exposed as tools in the agent's action space, and GRPO (Group Relative Policy Optimization) trains the optimal policy.

This approach outperforms hand-crafted heuristics on long-horizon benchmarks. The trade-off: it requires RL training infrastructure, produces model-specific policies, and the resulting behavior is non-deterministic.

### Capacity-Driven Storage

When the context window approaches capacity, compress and evict to external storage. This is a practical trigger — storage happens because the agent needs room, not because the content is inherently valuable. It works but is reactive rather than proactive, and storage quality depends entirely on the compression quality.

## The Design Pattern: Trigger-Agnostic Infrastructure

A memory infrastructure layer should support all five patterns without preferring one. The decision of when to call the store API belongs to the agent, the orchestrator, the RL policy, or the system event handler. The infrastructure handles embedding generation, typed record creation, indexing, and lifecycle management regardless of what triggered the write.

This separation is important because trigger strategies evolve faster than storage infrastructure. An organization might start with event-driven triggers, experiment with agent-initiated storage, and eventually adopt RL-trained policies — all without changing their storage layer.

## The Importance Question

What makes a memory worth keeping? Three approaches have emerged:

### LLM-Assessed Importance

Park et al. (2023) used an LLM to rate each memory's importance on a 1-10 scale at write time. "Eating breakfast" scores 2. "Getting promoted" scores 9. This captures semantic nuance — an LLM can recognize that certain information is significant in ways that rules cannot.

The trade-off: an LLM call per memory, non-deterministic scoring, and the assessment happens at write time when the full future significance of the information isn't yet known.

### Heuristic Rules

Importance is derived from metadata: record type, namespace, tags, outcome signals. A failure record in the incidents namespace gets high importance. A user preference gets moderate importance. This is fast, deterministic, and requires no LLM — but misses nuance.

### Importance Emerges from Usage

Don't set importance at write time. Let it emerge from access patterns: how often the memory is recalled, whether it led to successful outcomes, how recently it was used. A memory recalled 50 times that led to good outcomes IS important — by evidence, not prediction.

This mirrors human memory. You rarely decide in the moment what's worth remembering. Importance reveals itself over time through what you find yourself recalling.

**A practical pattern:** Use emergent importance as the primary signal, with an optional write-time hint that the agent can provide when it has strong reason to believe something is significant. The hint seeds the initial importance; usage patterns take over as the dominant signal.

## Memory Evolution

Most memory systems treat stored records as immutable — once written, a memory stays the same until explicitly updated or deleted. A-Mem (Xu et al., 2025) challenged this with a concept that has significant implications: **memories should evolve.**

### How A-Mem's Evolution Works

When a new memory is stored, the system:
1. Generates a structured note with keywords, tags, and contextual description
2. Searches existing memories for relevant connections
3. Establishes bidirectional links where meaningful similarities exist
4. **Updates the contextual representations of existing memories** based on the new information

The last point is the key innovation. When you store "service A migrated to Kubernetes last week," an existing memory about "service A's architecture" gets enriched with the migration context — without the agent explicitly requesting the update.

### Why Evolution Matters

Consider an operations agent that stores three separate memories over three months:
1. "Service payment-api sometimes returns 502 errors"
2. "Payment-api's connection pool maxes out under load"
3. "Payment-api OOM kills correlate with connection pool exhaustion"

Without evolution, these are three independent memories. With evolution, memory #1 gets enriched with the root cause from memories #2 and #3. When the agent later searches for "502 errors," it finds not just the symptom but the full causal chain — because the early memory evolved to include later understanding.

### Evolution as a Design Dimension

| Approach | Mechanism | Complexity |
|----------|-----------|------------|
| Immutable | Store once, never change | Simple. Requires explicit updates for corrections. |
| Supersession | New memory marks old as deprecated | Moderate. Handles contradictions. Old versions preserved. |
| Full evolution | Existing memories updated when related new memories arrive | Complex. Richer knowledge over time. Requires similarity detection and update logic. |

The choice depends on your system's requirements. Supersession is a practical middle ground — simpler than full evolution but handles the most common case (contradicted information) without losing history.

## What to Store: Content Decisions

For the operations agent scenario at the top of this chapter, the stored memory should be a **curated record** — concise, actionable, including root cause, resolution, and verification:

> "OOM kills in payment-service pods caused by JDBC connection pool leak. HikariCP maxPoolSize was unbounded. Fix: set maxPoolSize=50 in application.yaml. Verified via Grafana JVM Metrics dashboard. Resolution time: 45 minutes."

Not the raw investigation log (500 lines of kubectl output, Grafana screenshots, hypothesis testing). The log belongs in log storage. The memory is the distilled knowledge.

The agent — or a post-session summarization step — decides what to curate. The storage layer stores whatever it receives. This is the intelligence-infrastructure boundary at work.

## Key Takeaways

1. **Multiple trigger patterns exist**, each with different trade-offs. The storage layer should support all of them without coupling to any specific trigger strategy.

2. **Importance is best determined empirically** — through usage patterns — rather than predicted at write time. Optional write-time hints can seed the initial value.

3. **Memory evolution is an emerging pattern** worth watching. At minimum, support supersession (new memories can mark old ones as deprecated). Full A-Mem-style evolution is more powerful but adds complexity.

4. **Store curated knowledge, not raw data.** The memory should be concise and actionable. Raw investigation logs, metrics, and conversation transcripts belong in different storage systems.

5. **The trigger strategy and the storage layer should be independent.** Organizations should be able to change how they decide what to store without changing where and how it's stored.
