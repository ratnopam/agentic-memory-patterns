# Five Design Questions Every Agent Memory System Must Answer

Regardless of implementation, every agent memory system must navigate five fundamental design questions. These questions are independent of framework, backend, or deployment model. How you answer them defines your architecture.

This chapter introduces the questions. Chapters 4-6 explore the patterns that address them.

## Question 1: When Should an Agent Store to Memory?

This seems simple — "store when you learn something useful." But the trigger decision has profound implications for memory quality, completeness, and cost.

The literature reveals a spectrum of approaches:

| Approach | Who Decides | Risk |
|----------|-------------|------|
| Store everything | Nobody — automatic | Storage grows unbounded; retrieval must be excellent |
| Agent-initiated | The LLM | Agent sometimes forgets to save important information |
| Event-driven | The orchestrator | May miss important intermediate observations |
| RL-trained policy | Learned behavior | Requires training infrastructure; non-deterministic |

Park et al. (2023) demonstrated that storing everything and filtering at retrieval is surprisingly robust — it eliminates the "forgot to store" failure mode. AgeMem (Yu et al., 2026) showed that RL-trained policies that learn when to store, update, and discard outperform hand-crafted heuristics. Both are valid — one invests in retrieval, the other invests in training.

The infrastructure implication: the storage layer should support all trigger patterns without preferring one. The decision of when to store is an intelligence concern that sits above the storage layer.

## Question 2: What Should Be Stored, and How Should It Evolve?

Not all information has equal long-term value. The CoALA framework (Sumers et al., 2023) maps cognitive science memory types to agent systems:

| Type | What It Contains | Example |
|------|-----------------|---------|
| **Semantic** | Stable facts and knowledge | "This service uses gRPC on port 8443" |
| **Episodic** | Events with temporal context and outcomes | "Pod crashed at 3am, root cause: pool leak" |
| **Procedural** | Workflows, runbooks, how-to knowledge | "To fix OOM: 1. Check limits 2. Inspect heap" |

But storage is only half the question. A-Mem (Xu et al., 2025) introduced a crucial concept: **memory evolution.** When new memories arrive, they don't just add to the store — they can trigger updates to existing memories. An old memory about "service A connects to service B" might get enriched with "via port 8443, authenticated with mTLS" when a new related memory arrives.

This means the design question isn't just "what to store" but "how does stored knowledge change over time?" Immutable storage (write once, never update) is simpler but loses the ability for knowledge to mature. Mutable storage (memories evolve) is more powerful but introduces complexity around versioning and consistency.

## Question 3: How Do You Prevent Cross-Session Contamination?

Cross-session memory enables learning. But it also enables pollution — stale information, incorrect generalizations, and scope leakage between unrelated contexts.

| Risk | Example |
|------|---------|
| Stale information | Agent uses a credential that was rotated months ago |
| Incorrect generalization | Fix from environment A misapplied in environment B |
| Scope leakage | One tenant's data surfaces in another tenant's context |
| Contradictory memories | "Use approach X" and "Never use X" both stored |

The question is not whether to support cross-session memory, but how to make it safe. This involves session isolation by default, explicit promotion of validated knowledge to shared spaces, and lifecycle management that handles stale and contradictory information.

## Question 4: Where Is the Boundary Between Memory and RAG?

Memory and RAG are often treated as separate systems, but they exist on a spectrum:

```
Static/Human-authored ←────────────────────→ Dynamic/Agent-authored

Reference docs    Curated runbooks    Session summaries    Episodic memories
(pure RAG)        (procedural KB)     (hybrid zone)        (pure memory)
```

The fundamental differences:

| | RAG | Agent Memory |
|--|-----|-------------|
| **Authorship** | Human-written documents | Agent-generated records |
| **Mutability** | Read-only, updated externally | Read-write, agent manages |
| **Purpose** | Ground in reference material | Learn from experience |

But the line blurs when agents annotate runbooks with their experiences, when session summaries are stored in vector databases, or when curated documents are ingested as procedural memory.

Rather than enforcing a hard boundary, a practical approach is to support both through the same retrieval interface, differentiated by metadata — who authored it (human vs agent), whether it's mutable, and how it should age.

## Question 5: How Do You Score, Decay, and Manage Lifecycle?

Not all memories are equally valuable. A memory recalled 50 times that led to successful outcomes is more valuable than one stored six months ago and never accessed.

Park et al. (2023) proposed the most cited scoring function: **recency × importance × relevance.** Recent work (AgeMem, 2026) has shown that RL-trained scoring policies can outperform hand-crafted formulas. The trade-off is between:

- **Deterministic scoring:** Fast, cheap, explainable, reproducible. Good for infrastructure.
- **Learned scoring:** More accurate for task-specific domains. Requires training. Non-deterministic.

Beyond scoring, the full lifecycle question includes:
- **Decay:** Should memories fade? How fast? Power law (slow, preserves old knowledge) or exponential (fast, favors recency)?
- **Evolution:** Should memories update when new context arrives, or remain immutable?
- **Tiering:** Should hot (recent, frequent) and cold (old, rare) memories be stored differently?
- **Archival:** When should memories move to cheaper storage? Should they ever be permanently deleted?

No existing production system handles lifecycle management comprehensively. This remains the **largest unsolved operational problem** in agent memory.

## How the Questions Connect

These five questions form an interconnected design space:

```
Q1 (When to store) ──→ Q2 (What to store, how it evolves) ──→ Storage
                                                                  │
Q3 (Cross-session safety) ←──── Retrieval ←───────────────────────┤
                                                                  │
Q4 (Memory vs RAG boundary) ──→ Unified retrieval interface ──────┤
                                                                  │
Q5 (Scoring, decay, lifecycle) ←── Applied at retrieval + background ─┘
```

Your answers to these questions will determine the architecture. The following chapters explore the design patterns that address each one.
