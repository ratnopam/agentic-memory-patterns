# Introduction

While exploring how to add persistent memory to an operations agent — one that diagnoses Kubernetes incidents, runs kubectl commands, and checks dashboards — a frustrating pattern emerged. The agent was good at its job within a session. But every new session was a blank slate. The incident it resolved yesterday? Gone. The runbook it discovered worked? Forgotten. The user's preferred cluster? Ask again.

A smarter model doesn't help if the agent can't remember what it learned yesterday.

The natural next step was to look for a memory solution. What emerged was a fragmented landscape — dozens of tools, each making different assumptions about what memory should be, where it should live, and who should manage it. Some required LLM calls on every write. Some locked you into a specific framework. Some worked well on a laptop but had no story for Kubernetes. None of them cleanly separated the question of *how memories are created* from the question of *where they're stored and how they're found.*

This book is what I wish existed when I started thinking through the problem.

## What This Book Is

This is a book about thinking through agent memory before building it. The design space is larger than it first appears, and the choices you make early — what to store, how to score, whether memories evolve — cascade through the entire system.

This book doesn't prescribe a single "best" approach. After surveying the research, exploring multiple implementation paths, and deploying a working system in production, the honest conclusion is that the right architecture depends heavily on your constraints. What this book offers is a framework for thinking about those constraints clearly — the questions to ask, the patterns that have emerged, and the trade-offs that aren't obvious until you're deep in the design.

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
