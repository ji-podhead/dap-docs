# DAP RAG — Reference

RAG in DAP is not a tool agents call. It is a **workflow phase type** (`type: rag`) — grounding happens as a structured step with a token budget, access control, and graph persistence. Built on SurrealDB HNSW — no separate Qdrant needed for graph-linked collections.

## DAP vs MCP

| | MCP | DAP |
|---|---|---|
| How accessed | Tool call → raw chunk dump | `type: rag` workflow phase |
| Token cost | ~1,500 tokens (raw chunks) | ~400 tokens (budget-capped + summarized) |
| Access control | Custom middleware | SurrealDB PERMISSIONS automatic |
| Agent experience | Same for all agents | Skill artifacts injected alongside chunks |
| Persistence | Discarded | Graph-linked in SurrealDB |
| Graph + vector | Two queries + app join | Single SurrealDB query |

## SurrealDB HNSW Vector Search

```surql
-- Define vector index on any table
DEFINE FIELD embedding ON web_content TYPE array<float>;
DEFINE INDEX web_content_vec ON web_content
  FIELDS embedding HNSW DIMENSION 1536 DIST COSINE;

-- Query: vector search + ACL filter + graph-ready IDs
SELECT id, title, url,
       vector::similarity::cosine(embedding, $query_vec) AS score
FROM web_content
WHERE vector::similarity::cosine(embedding, $query_vec) > 0.75
  AND access_level IN $auth.access_levels     -- ACL automatic via PERMISSIONS
ORDER BY score DESC LIMIT 5;
```

## Graph + Vector in One Query

```surql
-- "Contacts who know about blockchain regulation" — no Qdrant + SurrealDB roundtrip
SELECT ->knows->agent.name AS contact,
       vector::similarity::cosine(->knows->agent.expertise_embedding, $q) AS score
FROM agent:alice
WHERE vector::similarity::cosine(->knows->agent.expertise_embedding, $q) > 0.7
ORDER BY score DESC LIMIT 5;
```

This is impossible with Qdrant alone — you'd need a two-step query + app-level join.

## `type: rag` Phase Config

```yaml
- id: ground_context
  type: rag
  source: surreal              # SurrealDB HNSW
  collections:
    - web_content_public
    - "agent_memory_{{ agent_id }}"
    - "skill_artifacts_{{ skill }}"
  query_from: task.input       # what to embed as query vector
  top_k: 5
  max_tokens: 400              # hard budget — no unbounded dumps
  summarize: true              # compress top_k chunks before injection
  persist_links: true          # RELATE agent->fetched->found_chunks
  access_filter: auto          # respects $auth.access_levels
  inject_as: grounding
```

## 4-Layer Access Control (Zero Extra Code)

```
Layer 1: Capabilities    --deny-arbitrary-query=record → only DEFINE API endpoints
Layer 2: SurrealDB RBAC  PERMISSIONS FOR select WHERE access_level IN $auth.access_levels
Layer 3: HNSW filter     payload: access_level IN $auth.access_levels (query time)
Layer 4: Casbin          /tools/rag_advanced/classified → role:clearance_3 only
```

An agent gets exactly the chunks they are allowed to see. No post-processing filter.

## Skill Artifacts as RAG Collections

The agent's skill store is a RAG collection. When an `llm` phase runs, it gets:
1. Web content chunks (external knowledge, budget-capped)
2. Skill artifacts (agent's accumulated approaches — same HNSW query)
3. Past memories (similar experiences from `agent_memory_{id}`)

Agent with `financial_analysis: 5` → only web chunks.
Agent with `financial_analysis: 75` → web chunks + 3 proven strategies from skill store.

## Persistence — Graph Linking

With `persist_links: true`:

```surql
-- After RAG phase, found chunks are graph-linked
RELATE agent:alice->fetched->web_content:["bun.sh", "changelog-1.2"]
  SET session_id = $session, score = 0.91, at = time::now();

RELATE web_content:["bun.sh", "changelog-1.2"]->supports->thesis:bun_handle_rename;
```

Future sessions can traverse: "what did I find before about this topic?" — one graph query, no re-search.

## Where to Store What

| Content | Store | Why |
|---|---|---|
| In-sim content (company pages, announcements) | SurrealDB full-text + HNSW | Graph-linked, PERMISSIONS automatic |
| Fetched web content metadata + graph | SurrealDB | URL records, RELATE edges |
| Web content text chunks | SurrealDB HNSW | Graph + vector in one query |
| External archive (millions of docs) | Qdrant | Scale-out only when SurrealDB insufficient |
| Agent memories | SurrealDB HNSW | Private to agent, graph-linked to sessions |
| Skill artifacts | SurrealDB HNSW | Inherited via employment graph |

---
> **References**
> - Lewis et al. (2020). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.* NeurIPS 2020. [arXiv:2005.11401](https://arxiv.org/abs/2005.11401)
> - Edge et al. (2024). *From Local to Global: A Graph RAG Approach to Query-Focused Summarization.* Microsoft Research. [arXiv:2404.16130](https://arxiv.org/abs/2404.16130)
> - Malkov & Yashunin (2018). *Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs.* IEEE TPAMI. [arXiv:1603.09320](https://arxiv.org/abs/1603.09320)

*Full spec: [dap_protocol.md §12b](../../planning/prd/dap_protocol.md)*
