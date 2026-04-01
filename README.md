# Agent Memory Patterns

*Architecture and Design for Persistent AI Agent Memory*

## Overview

A guide to the design decisions, architectural patterns, and open challenges of building persistent, searchable memory for AI agents. Covers memory formation, cross-session safety, scoring and decay, lifecycle management, and the design boundary between memory intelligence and memory infrastructure.

This book assumes familiarity with RAG, vector databases, and LLM agent architectures. It focuses on the design questions specific to agent memory — the decisions that shape how agents learn, retain, and retrieve knowledge over time.

## Download

**[Download PDF](https://github.com/ratnopam/agent-memory-patterns/releases/latest)** — generated automatically from the latest tagged release.

## Chapters

| # | Title | Key Question |
|---|-------|-------------|
| 1 | Why Context Windows Are Not Memory | Does scaling context eliminate the need for external memory? |
| 2 | The Memory Landscape | What architectural patterns exist and what's still unsolved? |
| 3 | Five Design Questions | The questions that define a memory system's architecture |
| 4 | Memory Formation and Evolution | Who decides what to store, and how should memories change over time? |
| 5 | Cross-Session Memory | How to share knowledge across sessions without contamination? |
| 6 | Scoring, Decay, and Lifecycle | How to rank memories, when to let them fade, and lifecycle management |
| 7 | Designing an Agentic Memory System | One opinionated approach: modular, pluggable, open-source |

## License

Book content: [CC BY-NC-ND 4.0](LICENSE)

## Author

Ratnopam Chakrabarti

## Feedback

Open an [issue](https://github.com/ratnopam/agent-memory-patterns/issues) for corrections, suggestions, or discussion.
