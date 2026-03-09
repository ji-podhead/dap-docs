# DAP IDE — Integration Reference

DAP IDE is a Vibe Coding tool for teams — a complete dev environment where humans and agents develop, review, and deploy as equals. It uses DAP for all agent tool discovery and invocation, SurrealDB as its single source of truth, and Qdrant for codebase semantic search.

> Not another agent framework — a complete environment where human decisions, agent reasoning, commits, PRs, and sprint state all live in the same graph.

**Full spec rendered below:** [`dap_ide.md`](../../planning/prd/dap_ide.md)

**DAP protocol touchpoints used by DAP IDE:**

| DAP Feature | DAP IDE Use |
|---|---|
| `DiscoverTools` | Agents discover task-appropriate tools — codebase context narrows results |
| `skill_min` gate | Task complexity → required skill score — prevents junior agents on senior tasks |
| Workflow `rag` phase | Codebase chunks + task graph injected before every LLM call |
| `PoT scoring` | Code review quality gate |
| `DAP Apps` async | Long-running agents via DAPQueue — no held gRPC connection |
| `DAP Logs` | Full audit trail — token cost, latency, outcome per tool call |
| Langfuse traces | Per-span observability on every LLM call |
| Haystack guardrails | Input: prompt injection; Output: code review completeness |

---

*See also: [dap-games.md](dap-games.md) · [apps.md](apps.md) · [tasks.md](tasks.md) · [observability.md](observability.md)*
