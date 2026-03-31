# Agent Memory Patterns

*Architecture and Design for Persistent AI Agent Memory*

## Overview

A practical guide to the design decisions, architectural patterns, and open challenges of giving AI agents persistent, searchable memory. Covers memory formation, evolution, cross-session safety, scoring and decay, lifecycle management, and the design boundary between memory infrastructure and memory intelligence.

Based on analysis of recent research (AgeMem, A-Mem, Generative Agents, MemGPT, CoALA), existing memory systems, and a working deployment on Kubernetes.

**This is NOT** an introduction to RAG, vector databases, or embeddings. It assumes familiarity with these concepts and focuses on the design questions specific to agent memory systems.

## Read the Book

- **[Read online](https://ratnopam.github.io/agent-memory-patterns/)** (GitHub Pages)
- **[Download PDF](https://github.com/ratnopam/agent-memory-patterns/releases)** (latest release)

## Chapters

| # | Title | Key Question |
|---|-------|-------------|
| 1 | Why Context Windows Are Not Memory | Does scaling context eliminate the need for external memory? |
| 2 | The Memory Landscape | What architectural patterns exist and what's still unsolved? |
| 3 | Five Design Questions | The questions that define an agent memory system's architecture |
| 4 | Memory Formation and Evolution | Who decides what to store, and how should memories change over time? |
| 5 | Cross-Session Memory | How to share knowledge across sessions without contamination? |
| 6 | Scoring, Decay, and Lifecycle | How to rank memories, when to let them fade, and the unsolved lifecycle problem |
| 7 | Designing an Agentic Memory System | One opinionated approach: modular, pluggable, open-source |
| 8 | Case Study | A real migration from key-value to semantic memory on Kubernetes |

## Reference Implementation

The design patterns in this book are grounded in a working implementation deployed on Kubernetes. A reference implementation is being prepared for open-source release and will be linked here when available.

## Key References

- Park et al., "Generative Agents" (UIST 2023) — memory stream + recency-relevance-importance scoring
- Packer et al., "MemGPT" (NeurIPS 2023) — OS-inspired virtual memory for agents
- Yu et al., "AgeMem" (2026) — RL-trained unified memory management
- Xu et al., "A-Mem" (2025) — Zettelkasten-inspired memory evolution
- Sumers et al., "CoALA" (2023) — cognitive architectures for language agents
- Zhang et al., "Memory Survey" (2024) — taxonomy of agent memory mechanisms

Full references with links in the [References chapter](src/references.md).

## License

Book content: CC BY-NC-ND 4.0

Code examples: Apache 2.0

## Author

Ratnopam Chakrabarti

## Feedback

Open an [issue](https://github.com/ratnopam/agent-memory-patterns/issues) for corrections, suggestions, or discussion.
