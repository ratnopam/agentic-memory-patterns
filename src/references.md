# References & Further Reading

## Research Papers Cited in This Book

### Foundational Work

- **Park et al.** (2023). "Generative Agents: Interactive Simulacra of Human Behavior." *UIST 2023*. [arxiv.org/abs/2304.03442](https://arxiv.org/abs/2304.03442)
  — Introduced the memory stream architecture and the recency-relevance-importance scoring function. The most cited work on agent memory.

- **Packer et al.** (2023). "MemGPT: Towards LLMs as Operating Systems." *NeurIPS 2023*. [arxiv.org/abs/2310.08560](https://arxiv.org/abs/2310.08560)
  — OS-inspired virtual memory hierarchy for agents. Core memory + archival memory with agent-managed paging.

- **Sumers et al.** (2023). "Cognitive Architectures for Language Agents." (CoALA). [arxiv.org/abs/2309.02427](https://arxiv.org/abs/2309.02427)
  — Framework mapping cognitive science memory models to LLM agent architectures. Defines working, episodic, semantic, and procedural memory.

- **Zhang et al.** (2024). "A Survey on the Memory Mechanism of Large Language Model based Agents." [arxiv.org/abs/2404.13501](https://arxiv.org/abs/2404.13501)
  — Comprehensive taxonomy of memory mechanisms across LLM agent systems.

- **Liu et al.** (2023). "Lost in the Middle: How Language Models Use Long Contexts." [arxiv.org/abs/2307.03172](https://arxiv.org/abs/2307.03172)
  — Demonstrated non-uniform attention in long contexts. Information in the middle is retrieved less reliably.

### Recent Advances (2025-2026)

- **Yu et al.** (2026). "Agentic Memory: Learning Unified Long-Term and Short-Term Memory Management for Large Language Model Agents." (AgeMem). [arxiv.org/abs/2601.01885](https://arxiv.org/abs/2601.01885)
  — RL-trained unified memory management. Demonstrates that learned memory policies outperform heuristics. Uses GRPO to train store/update/discard decisions.

- **Xu et al.** (2025). "A-Mem: Agentic Memory for LLM Agents." [arxiv.org/abs/2502.12110](https://arxiv.org/abs/2502.12110)
  — Zettelkasten-inspired interconnected knowledge networks. Memory evolution — existing memories updated when new context arrives. Bidirectional linking between related memories.

- **Behrouz et al.** (2024). "Titans: Learning to Memorize at Test Time." [arxiv.org/abs/2501.00663](https://arxiv.org/abs/2501.00663)
  — Neural long-term memory module built into the model architecture. Handles 2M+ token contexts. Represents parametric (in-model) memory, complementary to explicit external memory.

- **Rasmussen et al.** (2025). "Zep / Graphiti: Temporal Knowledge Graph Architecture for Agent Memory."
  — Temporal knowledge graph with valid_at/invalid_at timestamps for contradiction handling. Multi-signal retrieval combining BM25, embeddings, and graph traversal.

- **Hu et al.** (2025). "Memory in the Age of AI Agents: A Survey."
  — Updated taxonomy covering token-level, parametric, and latent memory forms. Highlights the shift toward agents that manage their own context.

- **Fang et al.** (2025). "MemP: Procedural Memory from Trajectories."
  — Distills agent trajectories into procedural abstractions for continual refinement and transfer.

- **Han et al.** (2025). "LEGOMem: Modular Procedural Memory for Multi-Agent Workflows."
  — Modular procedural memory design for multi-agent systems.

### Classic References

- **Ebbinghaus, H.** (1885). *Über das Gedächtnis* (On Memory).
  — Foundational work on memory decay. Established the forgetting curve (power law decay).

- **Miller, G.A.** (1956). "The Magical Number Seven, Plus or Minus Two." *Psychological Review*.
  — Working memory capacity limits.

## Production Systems Referenced

- **Mem0** — Open-source memory layer with LLM extraction. [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0)
- **Letta / MemGPT** — Agent framework with self-editing memory. [github.com/letta-ai/letta](https://github.com/letta-ai/letta)
- **Zep / Graphiti** — Temporal knowledge graph for agent memory. [github.com/getzep/graphiti](https://github.com/getzep/graphiti)
- **kagent** — Kubernetes-native AI agent framework. [github.com/kagent-dev/kagent](https://github.com/kagent-dev/kagent)
- **engram** — Pluggable memory infrastructure (reference implementation). [github.com/ratnopam/engram](https://github.com/ratnopam/engram)
- **pgvector** — Vector similarity search for PostgreSQL. [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)
