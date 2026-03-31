# Chapter 4: Memory Formation and Evolution

This chapter addresses two of the most consequential design decisions: when should an agent store to memory, and how should stored knowledge change over time.

## The Trigger Problem

A common early mistake when building agent memory: storing a record after every successful tool call — every kubectl output, every log snippet, every metric check. Within a day, the memory store has thousands of records. When the agent searches for "OOM kills," it gets back a wall of raw output fragments instead of the one concise record that actually captures the root cause and fix.

The problem isn't storing too much — research on generative agents proved that storing everything can work. The problem is storing the wrong *kind* of everything. Raw tool outputs aren't memories. A memory is the distilled knowledge: "OOM kills in payment-service caused by connection pool leak. Fix: set maxPoolSize=50." That distinction between raw data and curated knowledge turns out to be one of the most important design decisions in any memory system.

What should an agent store in long-term memory? The answer depends on the architecture's philosophy toward memory formation.

## Five Trigger Patterns

### Store Everything, Filter at Retrieval

The most aggressive approach, demonstrated in the Generative Agents work (2023): every observation becomes a memory record. Mundane events and significant events alike are stored. The sophistication is entirely in the retrieval scoring function — recency, relevance, and importance determine what surfaces when memories are searched.

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

## Determining What Matters: The Importance Question

What makes a memory worth keeping? Three distinct approaches have emerged, each with a different philosophy about when and how importance should be determined.

### Approach 1: LLM-Assessed Importance

The first approach uses an LLM to rate each memory's importance at write time, typically on a numerical scale. A mundane observation like "the user ate breakfast" might receive a low score, while "the database password was rotated" receives a high one. This captures semantic nuance effectively — an LLM can recognize significance in ways that simple rules cannot.

However, this approach carries trade-offs. It requires an LLM call for every memory stored, which adds latency and cost. The scoring is non-deterministic, meaning the same memory might receive different importance ratings on different runs. Most importantly, the assessment happens at write time, when the full future significance of the information is not yet known. A seemingly routine observation today might turn out to be the critical clue in next month's incident investigation.

### Approach 2: Heuristic Rules

The second approach derives importance from structured metadata rather than content analysis. A failure record stored in an incidents namespace automatically receives high importance. A user preference receives moderate importance. A routine log entry receives low importance. The rules are based on record type, namespace conventions, tags, and outcome signals.

This approach is fast, deterministic, and requires no LLM involvement. Its weakness is that it cannot assess importance based on the actual content of the memory — it relies entirely on metadata signals, which means it will miss cases where the content itself is significant but the metadata doesn't reflect that.

### Approach 3: Emergent Importance from Usage

The third approach takes a fundamentally different stance: rather than attempting to assess importance at write time, it allows importance to emerge naturally from how the memory is actually used. How often is the memory recalled? When it is recalled, does it lead to successful outcomes? How recently has it been accessed?

A memory that has been recalled fifty times and led to good outcomes in forty of those cases is demonstrably important — not because someone predicted it would be, but because usage patterns have proven it. Conversely, a memory that has never been recalled in six months, regardless of what importance score it received at write time, has not demonstrated its value.

This mirrors how human memory works. People rarely decide in the moment what is worth remembering. Importance reveals itself over time through what you find yourself recalling and what proves useful in practice.

### Combining the Three Approaches

In practice, the most robust pattern combines emergent importance as the primary signal with an optional write-time hint. The agent can provide an initial importance value when it has strong reason to believe something is significant — for example, when it resolves a critical incident or discovers a novel failure mode. This hint seeds the initial importance score. Over time, actual usage patterns take over as the dominant signal, either reinforcing the initial assessment or allowing it to fade if the memory proves less valuable than expected.

## A Note on Memory Evolution

Most memory systems treat stored records as immutable — once written, a memory remains unchanged until explicitly updated or deleted. Recent work, notably A-Mem (Xu et al., 2025), has explored an alternative: memories as living documents that evolve over time. In this approach, when a new memory is stored, the system identifies related existing memories and updates their context — enriching earlier knowledge with later understanding. For example, an early memory about "service A sometimes returns 502 errors" could be automatically enriched when a later memory identifies the root cause as connection pool exhaustion. This is a promising research direction, though it adds significant complexity to the storage layer. For most practical implementations, a simpler approach — supersession, where new memories can explicitly mark older ones as deprecated without modifying them — provides the most important benefit (handling contradictions) without the complexity of full evolution.

## What to Store: Content Decisions

For the operations agent scenario at the top of this chapter, the stored memory should be a **curated record** — concise, actionable, including root cause, resolution, and verification:

> "OOM kills in payment-service pods caused by JDBC connection pool leak. HikariCP maxPoolSize was unbounded. Fix: set maxPoolSize=50 in application.yaml. Verified via Grafana JVM Metrics dashboard. Resolution time: 45 minutes."

Not the raw investigation log (500 lines of kubectl output, Grafana screenshots, hypothesis testing). The log belongs in log storage. The memory is the distilled knowledge.

The agent — or a post-session summarization step — decides what to curate. The storage layer stores whatever it receives. This is the intelligence-infrastructure boundary at work.

## Key Takeaways

Multiple trigger patterns exist in the landscape, each with different trade-offs. A well-designed storage layer should support all of them without coupling to any specific trigger strategy, because the decision of when and what to store will evolve faster than the storage infrastructure.

Importance is best determined empirically through actual usage patterns rather than predicted at write time. Optional write-time hints can seed the initial value, but the system should allow usage evidence to take over as the primary signal.

Memory evolution — the idea that existing memories should be enriched as new context arrives — is an emerging pattern worth considering. At minimum, a memory system should support supersession, where new memories can mark older ones as deprecated. Full evolution, as demonstrated by A-Mem, is more powerful but adds complexity that may not be warranted in every deployment.

Finally, the distinction between curated knowledge and raw data is worth maintaining deliberately. Memories should be concise and actionable. Raw investigation logs, metrics, and conversation transcripts serve important purposes but belong in storage systems designed for those workloads, not in the memory store that agents search for operational knowledge.
