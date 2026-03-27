# Cross-Session Memory Without Contamination

Cross-session memory is what separates a knowledgeable agent from a stateless one. But sharing knowledge across sessions introduces real risks. This chapter examines the contamination problem, explores scoping patterns, and proposes the knowledge building pipeline as an architectural pattern.

## The Contamination Problem

When an agent recalls memories from previous sessions, failure modes emerge that don't exist in single-session systems:

| Risk | Scenario | Consequence |
|------|----------|-------------|
| **Stale information** | Agent recalls a configuration value changed months ago | Incorrect action based on outdated knowledge |
| **Incorrect generalization** | Solution from environment A applied to environment B with different constraints | Wrong remediation, potentially making things worse |
| **Scope leakage** | One tenant's operational data surfaces in another tenant's agent session | Data privacy violation |
| **Bias amplification** | Agent's early (poor) solutions bias all future sessions | Performance degrades rather than improves over time |
| **Contradictory memories** | "Use approach X" from session 1, "Never use X" from session 5 | Agent receives conflicting guidance, unpredictable behavior |

These are not theoretical. Any system that maintains knowledge across sessions will encounter them. The question is not whether to prevent them but how.

## The Isolation-Sharing Spectrum

At one extreme, every session is fully isolated — no cross-session memory at all. This is safe but means the agent never learns. At the other extreme, all memories are shared across all sessions — maximum learning but maximum contamination risk.

The practical middle ground: **session isolation by default, with explicit mechanisms for promoting validated knowledge to shared spaces.**

```
Per-session ←─────────────────────────→ Fully shared

Session-scoped    Namespace-scoped    Domain knowledge    Global
memory only       sharing             base                sharing
(safe, no learn)  (controlled)        (curated)           (risky)
```

Most production systems start on the left and carefully move rightward as they gain confidence in their curation mechanisms.

## Scoping Patterns

### Namespace-Based Scoping

Hierarchical string paths organize memories into logical boundaries:

```
/sessions/{session-id}/     ← per-session (isolated by default)
/knowledge/{domain}/        ← curated shared knowledge
/users/{user-id}/           ← per-user preferences
/agents/{agent-id}/         ← agent-private learnings
```

Search is scoped by namespace prefix. Searching `/sessions/abc123` returns only that session's memories. Searching `/knowledge/` returns all shared knowledge. The agent or orchestrator controls which namespaces to search.

**Strengths:** Simple to implement (string column with prefix matching). Intuitive hierarchy. Dynamic — no configuration needed to create namespaces. The same pattern works across backends.

**Limitations:** Convention-based — any agent CAN search any namespace. No enforcement without an access control layer.

### Graph-Based Scoping

Some systems (particularly those using knowledge graphs) scope access through graph relationships. An agent can traverse only the subgraph it has access to. Entities and relationships carry ownership and visibility attributes.

**Strengths:** Rich access control. Natural for entity-centric knowledge. Temporal scoping (what was true at time T).

**Limitations:** Requires a graph database. More complex to implement and reason about. Harder to migrate between backends.

### Tag and Metadata Scoping

Memories carry metadata tags (tenant_id, project_id, access_level), and retrieval filters on these tags. This is more flexible than namespaces but less structured.

**Strengths:** Flexible, multi-dimensional scoping. Works with any backend that supports metadata filtering.

**Limitations:** No implicit hierarchy. Tag conventions must be enforced externally.

## The Knowledge Building Pipeline

Rather than allowing raw cross-session memory access, an effective pattern interposes a **processing layer** between session memories and shared knowledge:

```
Session ends
    │
    ▼
Raw session memories (private, per-session namespace)
    │
    │  Processing layer (runs post-session or on schedule)
    │  ├─ Extracts reusable knowledge
    │  ├─ Deduplicates against existing shared knowledge
    │  ├─ Validates accuracy (LLM, rules, or human review)
    │  └─ Assigns to appropriate domain namespace
    │
    ▼
Domain knowledge base (shared namespace, cross-agent)
    │
    ▼
Future sessions search shared knowledge
```

**The processing layer sits above the storage infrastructure.** It reads raw session memories, applies intelligence (extraction, validation, deduplication), and writes curated knowledge back to a shared space. This separation means:

- Session isolation is maintained by default
- Only validated knowledge enters shared namespaces
- The processing can be an LLM pipeline, a rules engine, or a human review queue
- The storage layer doesn't need to know about the curation logic

This is also the pattern described as "sleep-time compute" in recent research — processing that happens between sessions to consolidate and organize knowledge, analogous to the hippocampal consolidation that occurs during human sleep.

## Handling Contradictions

When new information contradicts existing memories, the system needs a resolution strategy:

### Last-Write-Wins

The simplest approach. New information overwrites old. Easy to implement but loses history. If the new information is wrong, the correct old information is gone.

### Supersession

New memory explicitly marks the old one as deprecated:

```
New memory: "max_connections should be 200"
  └─ supersedes: old-memory-id
Old memory: "max_connections should be 50"
  └─ status: deprecated
  └─ superseded_by: new-memory-id
```

Both versions are preserved. The old memory is hidden from default search results but available for audit and history. This is the temporal edge pattern used in graph-based approaches, simplified for flat storage.

### Temporal Validity

Each fact has `valid_from` and `valid_until` timestamps. Queries default to "currently valid" but can ask for historical state. This is the most principled approach — Zep's Graphiti (Rasmussen et al., 2025) implements this in its temporal knowledge graph.

### The Practical Choice

Supersession is the right starting point for most systems. It handles the most common case (corrected information) without losing history, and it doesn't require the complexity of full temporal validity tracking.

## Lifecycle States

Memories should move through defined states, not just exist or not exist:

```
ACTIVE ──→ DEPRECATED ──→ EXPIRED ──→ ARCHIVED
  │              │
  │              └── superseded by newer memory
  │
  └── TTL passed without access renewal
```

- **Active:** Appears in search results with full scoring
- **Deprecated:** Superseded by newer information. Hidden from default results but preserved for history.
- **Expired:** TTL passed without being accessed. Not returned by default. Available for audit.
- **Archived:** Moved to cold storage. Batch access only.

The transition logic is infrastructure — deterministic rules based on TTL, access patterns, and explicit supersession. The decision to supersede is intelligence — made by the agent, the orchestrator, or a processing pipeline.

## Key Takeaways

1. **Default to session isolation.** Per-session memory is safe. Cross-session knowledge should flow through an explicit promotion mechanism — not be universally accessible by default.

2. **The knowledge building pipeline** is an effective pattern for safe cross-session sharing. Session memories are private. A processing layer extracts, validates, and promotes reusable knowledge to shared namespaces.

3. **Support supersession** for contradiction handling. New memories should be able to mark old ones as deprecated, preserving history while preventing stale information from surfacing.

4. **Implement lifecycle states** as infrastructure. Active, deprecated, expired, archived — each with different retrieval behavior. The storage layer manages transitions; the intelligence layer decides when to trigger them.

5. **Never hard-delete memories.** Storage is inexpensive. Lost information is irreplaceable. Even "wrong" memories have audit value. Soft-delete to archived, not permanent deletion.
