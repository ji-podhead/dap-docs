# DAP Bloat Score — Reference

The bloat score is a per-tool token cost metric and first-class protocol field in DAP. It measures and controls how many tokens a tool injects into agent context, ensuring discovery and invocation stay lean.

## Structure

Every tool, skill artifact, and workflow has a `bloat_score` computed at registration time:

```python
bloat_score = {
    "description_tokens":   18,    # tool name + one-line description
    "schema_tokens":        94,    # full parameter schema (loaded via GetToolSchema)
    "artifact_tokens":      340,   # tokens injected by artifact_binding (if any)
    "example_tokens":       0,     # example invocations (optional)
    "total":                452,   # sum — full context injection cost
    "summary_tokens":       18,    # what DiscoverTools injects (description only)
}
```

Each layer loads independently — the agent controls when each is added to context:
- `DiscoverTools` → injects only `summary_tokens` per tool
- `GetToolSchema` → injects `schema_tokens`
- Artifact binding → injects `artifact_tokens`

## Grades

| Grade | Criteria | Action |
|---|---|---|
| **A** (lean) | total <= 50 tokens | Preferred in discovery ranking |
| **B** (acceptable) | total <= 200 tokens | Normal operation |
| **C** (verbose) | total <= 500 tokens | Warning at registration |
| **D** (rejected) | total > 500 tokens | Rejected at registration — must be refactored |

## Discovery Ranking Formula

Bloat is a weighted factor in the discovery ranking alongside semantic relevance and success rate:

```
tool_rank = semantic_similarity * 0.55
          + success_rate         * 0.25
          + (1 - bloat_weight)   * 0.20

bloat_weight = normalize(summary_tokens, 0, 200)
```

A 10-token tool description ranks higher than a 150-token one at equal relevance and success rate.

## Cascading Budget

The agent runtime tracks total context usage across all sources:

```
activation_bundle + discovered_tools + injected_artifacts + conversation_history
```

When a new tool is requested, it only loads if it fits:

```python
if tool.bloat_score.total <= max_tokens_budget - current_usage:
    load_tool(tool)
```

Artifact selection also respects the budget:

```python
artifacts = await qdrant.search(
    collection=f"skill_{skill_name}_{agent_id}",
    vector=query_embedding,
    limit=top_k,
    filter={
        "must": [{"key": "injection_tokens", "range": {"lte": remaining_token_budget}}]
    }
)
```

## MCP vs DAP — Real Numbers

| Scenario | MCP | DAP |
|---|---|---|
| 50 tools available, 3 used | ~8,000 tokens (all 50 loaded) | ~54 tokens (3 summaries x 18) |
| Tool schema loaded | Always at session start | On demand via GetToolSchema |
| Artifact injection | Not supported | Only matching artifacts, within budget |
| Per-session overhead | 8,000+ tokens (constant) | 54-500 tokens (scales with actual use) |

Over 100 activation cycles with 50 tools: MCP costs ~800,000 tokens in tool-loading overhead. DAP costs ~5,000-50,000 tokens depending on task complexity.

## Bloat Efficiency in DAP Bench

Family A (Discovery Quality) includes a `bloat_efficiency` dimension:

```
bloat_efficiency = 1 - (actual_tokens_injected / task_completion_tokens_minimum)
```

A tool injecting 800 tokens to answer a 50-token question has low bloat efficiency. High-bloat tools are down-ranked in discovery unless their success rate justifies the cost.

## Token Cost in Proof Systems

| Proof type | Bloat source | Tracked as |
|---|---|---|
| PoS (Proof of Search) | Search results injected into reasoning context | `score.token_efficiency` |
| PoT (Proof of Thought) | Evidence and reasoning chain loaded for scoring | `pot_bloat_tokens` |
| PoD (Proof of Deliverable) | Certificate size attached to deliverable | `pod_size_bytes` (~300 bytes, negligible) |

## Skill Artifact Bloat

Bloat tracking extends to skill artifacts and workflows:

```yaml
artifact:
  type: workflow
  name: market_research_workflow
  quality_score: 0.84
  proofed: true
  bloat:
    workflow_tokens: 280
    injected_as: prepend_prompt
    injection_tokens: 280
```

The bloat score turns context efficiency from a design aspiration into a measurable, enforceable protocol property.

> **References**
> - Hoffmann et al., "Training Compute-Optimal Large Language Models" (Chinchilla paper) — token efficiency fundamentals
> - [Qdrant Filtered Search](https://qdrant.tech/documentation/concepts/filtering/) — budget-aware artifact selection

*Full spec: [dap_protocol.md §7](../../planning/prd/dap_protocol.md)*
