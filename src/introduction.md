# Introduction

Every AI agent built today has amnesia.

Your operations agent diagnoses a production incident, traces the root cause, applies a fix — and next week, when the exact same pattern occurs, it starts from scratch. Same investigation steps. Same compute cost. It learned nothing.

This isn't a failure of any particular framework or service. It's a missing layer. Agent frameworks excel at reasoning, tool use, and orchestration. But persistent, searchable, cross-session memory — the kind that lets an agent accumulate expertise over time — is still an open architectural problem.

## What This Book Is

This is a design and architecture book about **agent memory as a problem domain**. It does not prescribe a single "best" approach. Instead, it:

- Examines the fundamental challenges that make agent memory hard
- Surveys the architectural patterns that have emerged across research and production systems
- Identifies open questions and design trade-offs that every memory system must navigate
- Proposes design principles for building modular, extensible memory infrastructure
- Illustrates one implementation path as a reference (not the only path)

The goal is to help you think clearly about agent memory before committing to a particular implementation. The design space is larger than it first appears, and the choices you make early — what to store, how to score, whether memories evolve — cascade through the entire system.

## What This Book Does NOT Cover

This book assumes you understand:
- **Retrieval-Augmented Generation (RAG)** — fetching relevant documents to ground LLM responses
- **Vector databases and embeddings** — converting text to vectors and searching by similarity
- **LLM agent architectures** — how agents use tools, maintain state, and reason
- **Kubernetes basics** (for later chapters on deployment)

If you're building agents and wondering "how should I think about memory?" — this book is for you. If you're wondering "what is an agent?" — start with the foundational resources listed in the References chapter.

## The Central Observation

The agent memory landscape today conflates two distinct concerns:

```
┌─────────────────────────────────────────────────────────┐
│  INTELLIGENCE                                            │
│  How memories are created, curated, and evolved          │
│                                                          │
│  LLM-powered extraction, summarization, consolidation,  │
│  reinforcement learning, self-editing memory blocks      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE                                          │
│  Where memories live, how they're found, how they age    │
│                                                          │
│  Storage backends, indexing, retrieval, scoring,          │
│  lifecycle management, multi-agent sharing               │
└─────────────────────────────────────────────────────────┘
```

Most solutions bundle these layers together. This coupling means choosing an intelligence approach (how memories are extracted) also locks you into a storage approach (where they're kept), a deployment model (where it runs), and a framework (what agents can use it).

This book argues that understanding the boundary between these layers — and designing for it — is essential for building memory systems that are modular, portable, and long-lived. But it also acknowledges that the most effective systems, as recent research demonstrates, find intelligent ways to bridge these layers rather than treating them as entirely independent.

## How This Book Is Organized

| Chapter | Focus |
|---------|-------|
| **1. Why Context Windows Are Not Memory** | The fundamental difference between context and memory |
| **2. The Memory Landscape** | Architectural patterns, common approaches, and what's still unsolved |
| **3. Five Design Questions** | The questions that define an agent memory system's architecture |
| **4. Memory Formation and Evolution** | When to store, what makes something worth keeping, and how memories change over time |
| **5. Cross-Session Memory** | Session isolation, contamination risks, and knowledge building patterns |
| **6. Scoring, Decay, and Lifecycle** | How to rank memories, when to let them fade, and managing the full lifecycle |
| **7. Designing a Memory Infrastructure Layer** | One opinionated approach: modular, pluggable, open-source |
| **8. Case Study** | A real migration from key-value storage to semantic memory on Kubernetes |

Chapters 1-2 set the context. Chapters 3-6 are the core — the design questions and patterns that apply regardless of implementation. Chapters 7-8 show one way to implement them, with specific engineering choices and trade-offs made explicit.
