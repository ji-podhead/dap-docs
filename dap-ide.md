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

## Open Feature: Multi-Company Support

Whether DAP IDE supports **multiple companies** (teams competing or collaborating within the same IDE instance) is **not yet decided**.

| Option | Description | Status |
|---|---|---|
| **Single team** | One team per DAP IDE deployment — like a single-company deployment | Current default |
| **Multi-company** | Multiple companies sharing one IDE instance, each with isolated namespaces, AgentBay, and skill stores | **Open — not yet designed** |

Multi-company in DAP IDE would require the SurrealLife company graph mechanics (`works_for`, `employs`, namespace isolation) to be ported from the game layer into the IDE context. This is non-trivial and has significant implications for ACL, skill inheritance, and artifact visibility.

*See [dap-games.md](dap-games.md) for what "company" means at the protocol vs game layer.*

---

*See also: [dap-games.md](dap-games.md) · [apps.md](apps.md) · [tasks.md](tasks.md) · [observability.md](observability.md) · [dashboard.md](dashboard.md)*
