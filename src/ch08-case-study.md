# Chapter 8: From Key-Value to Semantic Memory on Kubernetes

This chapter describes a real migration: replacing key-value memory with vector-indexed semantic memory on a Kubernetes-native agent platform. The goal is not to promote a specific tool but to illustrate the practical decisions, trade-offs, and surprises that arise when adding persistent memory to a production agent.

## The Starting Point

The agent is an operations assistant deployed on a Kubernetes cluster. It helps operators manage infrastructure — diagnosing issues, running commands, checking metrics, and applying fixes. Built with LangGraph, deployed as a BYO (Bring Your Own) container agent on an agent management platform.

**Original memory: key-value store**

The agent's memory was a simple key-value implementation using an in-cluster store. Three operations: set defaults, get defaults, clear defaults. The stored data was a JSON blob per user:

```json
{"cluster": "prod-cluster", "namespace": "default"}
```

**What this could do:**
- Remember which cluster the user works on
- Remember their preferred namespace

**What this could NOT do:**
- Remember what incidents occurred and how they were resolved
- Search for similar past issues by meaning
- Share knowledge between agents
- Track which approaches worked and which failed

## The Migration

### Step 1: Choose the Storage Layer

Requirements:
- Must work on Kubernetes (Helm chart, ConfigMap-driven)
- Must use existing infrastructure (the cluster already had PostgreSQL)
- Must support semantic search (vector similarity, not just key lookup)
- Must be framework-agnostic (the agent is LangGraph today but may change)
- Must not add LLM calls to the critical path (cost and latency constraints)

The choice: a Python memory module with a pgvector backend, using the cluster's existing PostgreSQL database. One `CREATE EXTENSION vector` command and a table creation — zero new infrastructure.

### Step 2: Preserve Backward Compatibility

The original three tools (set/get/clear defaults) kept their exact signatures. Users who relied on "set my default cluster" saw no change. Five new tools were added alongside:

| Tool | Capability Added |
|------|-----------------|
| `remember_knowledge` | Store operational knowledge with typed records |
| `recall_knowledge` | Search memories by semantic similarity |
| `recall_context` | Composite retrieval (episodes + procedures + failures) |
| `mark_memory_outcome` | Record whether a recalled memory was helpful |
| `memory_stats` | Introspect what the agent knows |

The agent went from 3 memory tools to 8, from 23 total tools to 28. The additional tools are available but not required — the agent calls them when its reasoning determines they're useful.

### Step 3: Deploy the Infrastructure

A Helm chart runs a migration job that:
1. Enables the pgvector extension on the existing PostgreSQL
2. Creates the `agent_memory` table with vector column (1024 dimensions)
3. Creates HNSW index for cosine similarity search
4. Creates namespace prefix index for scoped queries
5. Creates JSONB metadata index for filtered search
6. Creates scoring columns (access_count, success_count, failure_count)

The migration is idempotent (`CREATE IF NOT EXISTS`) and runs as a post-install/post-upgrade Helm hook. Upgrading the memory infrastructure is a `helm upgrade` with no downtime.

### Step 4: Wire Embeddings

The agent runs on a cloud Kubernetes cluster with IAM role-based access. Embeddings use a cloud-hosted embedding model (1024 dimensions) authenticated via the pod's service account — no API keys in the configuration, no secrets to manage.

For teams running on-premises or air-gapped, the same module supports local embedding models (Ollama) with a configuration change. The agent code doesn't change.

## The Result

### Before: Key-Value (3 tools)

```
User: "What do you know about our database setup?"
Agent: "I don't have information about your database.
        Could you tell me which cluster to check?"
```

### After: Semantic Memory (8 tools)

```
User: "Remember that this cluster uses Aurora PostgreSQL 15.8
       with pgvector enabled for agent memory"
Agent: [stores semantic memory with metadata]

[Agent pod restarts — simulating new deployment]

User: "What do you know about our database setup?"
Agent: [searches memory by meaning → finds stored knowledge]
       [enriches with live cluster data from operations tools]

       "Based on my records and current cluster state:
        - Database: Aurora PostgreSQL 15.8
        - Extensions: pgvector enabled
        - Purpose: Vector database for agent memory
        [+ live infrastructure details]"
```

The agent combined recalled memory with live data — impossible with key-value.

### Memory Survives Pod Restarts

The critical validation: memory persists across pod lifecycle events.

```bash
kubectl rollout restart deployment/agent -n platform
# Pod recreated, new process, fresh context window

User: "What are my defaults?"
Agent: "Your defaults are: cluster=prod-cluster"
# Retrieved from PostgreSQL, not in-process state
```

This works because the memory module stores data in an external database, not in the agent's process memory. Key-value stores would also survive restarts — the added value is semantic search, typed records, and scoring.

## Architecture Change

The architecture shift was straightforward. Before the migration, the agent pod connected to a key-value store for three exact-match operations. After the migration, the agent pod imports a memory module (a Python library) that connects to PostgreSQL with the pgvector extension. The database holds an `agent_memory` table with an HNSW vector index for cosine similarity search, a namespace prefix index, JSONB metadata indexes, and scoring columns for tracking access patterns. Embedding generation is handled by a cloud embedding API, authenticated through the pod's service account.

The key-value store dependency was removed entirely. What was added: the memory module as a Python library in the container, the pgvector extension on the existing database, and embedding API calls for converting text to vectors.

**What stayed the same:** The agent's graph structure, its operations tools, its deployment pipeline, its Kubernetes CRD.

## Lessons Learned

### What Worked

1. **Lazy initialization** was essential. The memory module connects to the database on first tool call, not at process startup. This avoids event loop conflicts with the async agent framework and means the agent starts fast even if the database is momentarily unreachable.

2. **Stable record IDs** for user defaults use upsert semantics — the same record is overwritten, not duplicated. This preserves the key-value behavior for preferences while storing them alongside semantic memories.

3. **Using the platform's existing database** avoided new infrastructure. The migration job adds one extension and one table. No new database to provision, back up, or monitor.

4. **The wheel distribution model** (building the memory module as a Python wheel, copying into the Docker build context) kept the agent's dependency chain clean and avoided authentication issues with private repositories during Docker builds.

### What Surprised Us

1. **Auth compatibility matters.** The OpenSearch client library's SigV4 authentication required a specific HTTP connection class that wasn't the default. A one-line fix, but not documented. Testing against real infrastructure (not just mocks) caught this.

2. **Build scripts override Helm values.** The deployment script used `--set-json` which overrode the values file. Environment variables (like the database connection string) had to be injected by the script from Kubernetes secrets, not from static values files.

3. **Database version availability changes.** The infrastructure-as-code specified a database engine version that had been deprecated by the cloud provider. This required updating the version constraint — a reminder that infrastructure modules should not hardcode specific versions.

4. **The agent uses memory tools naturally.** With well-written tool descriptions, the LLM agent decides on its own when to store knowledge and when to search memory. No explicit prompting like "check your memory first" was needed — the tool descriptions ("search long-term memory for relevant past knowledge") guided the agent's reasoning.

## Key Takeaways

1. **The migration was additive.** Existing tools kept working. New tools were added alongside. No breaking changes. This is how memory should be introduced — incrementally, not as a rewrite.

2. **Memory infrastructure is invisible to the agent's core logic.** The agent's graph structure, its reasoning patterns, and its operations tools didn't change. Memory tools are additional capabilities, not a different architecture.

3. **The value is in search, not storage.** A key-value store can hold the same data. The difference is that semantic search finds memories by meaning — "what do you know about database issues?" matches a memory about "OOM kills caused by connection pool leak" even though the words don't overlap.

4. **Test against real infrastructure.** Auth mechanisms, version constraints, and deployment pipeline interactions are invisible in unit tests. Integration tests against actual databases and cloud services are essential for production memory systems.

5. **Start simple, add sophistication incrementally.** The first deployment used basic vector search. Scoring, composite retrieval, and lifecycle management can be added in subsequent iterations without changing the deployment model or the agent's code.
