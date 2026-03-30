# The Memory Landscape: What Exists and What's Missing

As of early 2026, agent memory is an active and rapidly evolving area. Multiple architectural patterns have emerged, each reflecting different design philosophies about what memory should be and how it should work. Understanding these patterns — their strengths, limitations, and underlying assumptions — is essential for making informed architectural choices.

## Architectural Patterns

Rather than surveying individual tools, this chapter examines the **patterns** that define the landscape. Every agent memory system embodies one or more of these.

### Pattern 1: Managed Extraction Pipeline

A managed service receives raw conversation events, runs LLM-powered extraction strategies (fact extraction, preference detection, session summarization, episodic structuring), and produces searchable memory records.

**Characteristics:**
- Intelligence and storage are bundled as one service
- Zero infrastructure management for the user
- LLM on the critical path for every memory operation
- Storage is typically managed and opaque

**Strengths:** Minimal setup. Sophisticated extraction without custom development. Well-suited for teams that want memory without managing infrastructure.

**Limitations:** Tied to a specific cloud provider. Limited control over storage and retrieval internals. Conversation-centric input model — not all agent data is a conversation.

### Pattern 2: Extract-Update-Search

An open-source library that, when called, sends content through an LLM to extract atomic facts, compares them against existing memories for deduplication and contradiction resolution, and stores the results in a vector database.

**Characteristics:**
- Every write triggers LLM extraction
- Memories are mutable — new information updates existing records
- Deduplication and conflict resolution via LLM judgment
- Multiple vector store backends supported

**Strengths:** Smart deduplication keeps memory clean. Memories evolve as understanding changes. Broad backend support.

**Limitations:** LLM required on every write path. Extraction quality depends on prompt engineering. The intelligence layer and storage layer are coupled — you must use the extraction to use the storage.

### Pattern 3: Agent Self-Management

The agent itself has explicit memory tools (append, replace, search, insert) and decides what to remember and when, as part of its reasoning process. Inspired by the MemGPT paper (Packer et al., 2023), which modeled memory as a virtual memory hierarchy — fixed-size "core memory" always in context, plus unlimited "archival memory" searched on demand.

**Characteristics:**
- The agent IS the memory manager
- Memory operations compete with other actions in the agent's reasoning
- Two-tier: working memory (always present) + archival (searched on demand)
- Framework-locked — tightly coupled to the agent runtime

**Strengths:** Elegant design. The agent has full context to make memory decisions. Natural mapping to the OS virtual memory paradigm.

**Limitations:** LLMs sometimes forget to save important information. Non-deterministic — same situation may or may not trigger storage. Tied to a specific agent framework.

### Pattern 4: Temporal Knowledge Graph

Conversations and documents are processed through a multi-step LLM pipeline: entity extraction, entity resolution, relationship extraction, contradiction detection, and graph construction. The result is a knowledge graph where facts have temporal validity.

**Characteristics:**
- Every fact is an entity-relationship triple with timestamps
- Contradictions are handled via temporal edges (old facts invalidated, not deleted)
- Rich relational queries via graph traversal
- Requires a graph database

**Strengths:** Most principled approach to contradiction handling. Temporal reasoning (what was true at time T). Rich relational queries.

**Limitations:** Requires a graph database (operational complexity). Multiple LLM calls per write for extraction and resolution. Graph schemas are less flexible than unstructured vector storage.

### Pattern 5: Reinforcement Learning for Memory Operations

A recent and significant development: training the agent's memory management policy via reinforcement learning. AgeMem (Yu et al., 2026) exposes memory operations (store, update, delete, retrieve, summarize, filter) as tools and trains the agent to use them optimally using GRPO (Group Relative Policy Optimization).

**Characteristics:**
- Memory operations are part of the agent's action space
- RL training learns WHEN and WHAT to store/update/discard
- Unified management of both short-term and long-term memory
- Outperforms heuristic-based approaches on benchmarks

**Strengths:** Empirically superior task performance. Adaptive — learns policies specific to the task domain. Unified STM+LTM management avoids the "independent optimization" problem.

**Limitations:** Requires RL training infrastructure. Policies are model-specific and non-deterministic. Training is compute-intensive. Not yet widely deployed in production systems.

### Pattern 6: Interconnected Knowledge Networks

A-Mem (Xu et al., 2025) introduced a Zettelkasten-inspired approach where memories are structured notes with keywords, tags, and contextual descriptions. When new memories arrive, the system identifies relevant connections to existing memories and establishes bidirectional links. Crucially, **existing memories evolve** — their attributes and context are updated as new related memories arrive.

**Characteristics:**
- Memories are richly structured notes, not flat records
- Bidirectional linking between related memories
- Memory evolution — old memories are updated when new context arrives
- Inspired by the Zettelkasten method for personal knowledge management

**Strengths:** Knowledge networks grow organically. Memory evolution keeps old knowledge fresh. Links enable traversal-based retrieval alongside vector search.

**Limitations:** Linking and evolution require computation on every write. Link quality depends on similarity thresholds. The network can grow complex and hard to debug.

## The Intelligence-Infrastructure Spectrum

These patterns exist on a spectrum:

```
More Intelligence ←──────────────────────────→ More Infrastructure

RL-trained     Extract-update    Agent self-     Knowledge      Pure storage
policies       pipeline          management      graph          + retrieval
(AgeMem)       (Pattern 2)       (MemGPT)        (Pattern 4)    (Pattern 7?)
```

The left side bundles intelligence with storage — the system decides what to remember, how to organize it, and when to forget. The right side separates them — the storage layer handles persistence, indexing, and retrieval, while intelligence decisions are made by the agent or an external pipeline.

Neither end is "correct." The choice depends on your constraints:

| If you need... | Lean toward... |
|----------------|---------------|
| Maximum task performance | More intelligence (RL-trained, extraction pipelines) |
| Explainability and auditability | More infrastructure (deterministic scoring, explicit storage) |
| Framework portability | More infrastructure (no framework coupling) |
| Minimal setup | Managed services (Pattern 1) |
| Full control over data | Infrastructure layer you own |
| Multi-agent sharing | Explicit storage with scoping mechanisms |

## What's Still Unsolved

Despite the rapid progress, several fundamental problems remain open across all approaches:

### Memory Lifecycle Management

Every system either accumulates memories indefinitely or relies on manual cleanup. No production system implements principled lifecycle management — automatic scoring, decay, demotion, and archival based on usage patterns. This is the single largest operational gap.

### Memory Evolution

Most systems treat memories as immutable after creation. A-Mem (Xu et al., 2025) demonstrated that memories should be living documents — updated, enriched, and relinked as new context arrives. This concept is not yet widely implemented.

### Unified Short-Term and Long-Term Management

AgeMem (Yu et al., 2026) showed that managing STM and LTM independently is suboptimal — what you keep in context affects what you need from long-term memory, and vice versa. Most production systems still treat these as separate concerns handled by different tools.

### Cross-Framework Memory Portability

Agent memory systems are typically locked to their framework. Memory accumulated in one framework cannot be used in another. A framework-agnostic memory layer that works across LangGraph, CrewAI, Google ADK, and others remains an open gap.

### Cost and Latency at Scale

Systems that use LLMs for extraction, deduplication, and scoring add latency and cost to every memory operation. At scale (millions of memories, thousands of daily operations), this becomes a significant engineering constraint. Deterministic alternatives trade accuracy for scalability — the optimal balance is not yet established.

## Design Space Summary

When designing an agent memory system, you're navigating this design space:

```
What triggers storage?
  └─ Everything (Park et al.) vs Agent-initiated (MemGPT) vs RL-trained (AgeMem)

What gets stored?
  └─ Raw events vs Extracted facts vs Structured notes vs Graph triples

How are memories organized?
  └─ Flat vector store vs Namespaced hierarchy vs Interconnected network vs Graph

How are memories found?
  └─ Cosine similarity vs Hybrid search vs Graph traversal vs RL-learned retrieval

How do memories age?
  └─ Never expire vs Hard TTL vs Soft decay vs RL-learned discard

How do memories evolve?
  └─ Immutable vs Supersession vs A-Mem evolution vs RL-learned update

Who manages memory?
  └─ The agent vs The orchestrator vs A managed service vs RL-trained policy

How is storage architected?
  └─ Single backend vs Multi-backend composition (key-value + search engine)
```

One dimension that deserves special attention is **storage architecture.** Most systems use a single backend for all memory operations. But production deployments increasingly reveal that different access patterns — fast session lookups, semantic similarity search, analytics — are best served by different storage engines working together. The multi-backend composition pattern (a fast key-value store as the source of truth, with a search engine as a derived semantic index) is emerging as a best practice for production-scale agent memory. Chapter 7 explores this in detail.

The chapters that follow explore each of these design dimensions in depth, examining the trade-offs and patterns that inform good architectural choices.
