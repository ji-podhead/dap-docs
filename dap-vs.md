# DAP vs — Comparison Reference

How DAP compares to the major alternatives: MCP (Model Context Protocol), Claude Code, and general LLM assistant architectures.

---

## DAP vs MCP

MCP and DAP solve different problems. They are **complementary, not competing**.

> MCP: connect a developer's LLM assistant to their local tools.
> DAP: give a fleet of autonomous agents access to an evolving, identity-aware, access-controlled tool ecosystem.

| Capability | MCP | DAP |
|---|---|---|
| **Tool set** | Fixed at session start | Dynamic — changes with ACL, skill tier, live registrations |
| **Discovery** | All schemas listed in system prompt | Live gRPC query at each activation, within token budget |
| **Access control** | Not built in | Casbin ACL is part of the protocol |
| **Tool search** | None | Semantic HNSW search filtered by ACL + skill |
| **Streaming** | Not native | gRPC native streaming |
| **Multi-tenancy** | Single agent | Fleet of agents — each sees different tool sets |
| **Dynamic registration** | Requires session restart | Index version bump → auto re-discover |
| **Context efficiency** | All tools in prompt (~10k tokens) | `max_tools` budget, lazy schema fetch (~900 tokens) |
| **Audit log** | External | Built into every InvokeTool call |
| **Skill gating** | None | First-class — tool invisible if skill below threshold |
| **RAG** | Tool call → raw chunk dump | `type: rag` phase — budget-capped, ACL-filtered, graph-linked |
| **Quality gate** | None | PoT threshold — retry or fail before delivery |
| **Anti-hallucination** | None | PoS — Z3-verified evidence chain |
| **Memory persistence** | Session ends → gone | Graph-linked in SurrealDB, retrievable across sessions |
| **Agent experience** | Same for all agents | Skill artifacts accumulated — better agents get richer context |

### Token Cost (same task)

```
MCP:
  50 tool schemas in system prompt       →  8,000 tokens
  RAG: 5 chunks × 300 tokens            →  1,500 tokens
  ────────────────────────────────────────────────────
  Total before agent does anything       → ~10,000 tokens
  Per-agent context differentiation     →       0 tokens

DAP:
  DiscoverTools: 4 tools × 10 tokens    →     40 tokens
  RAG phase: 5 chunks summarized        →    200 tokens
  Skill artifacts (experienced agent)   →    180 tokens
  LLM phase total context               →   ~600 tokens
  ────────────────────────────────────────────────────
  Total                                 →    ~900 tokens
  Per-agent context differentiation     → yes — artifacts vary by skill
```

### What Each Solves

```
MCP:  Agent → [static tool list] → tool() → raw chunks → answer

DAP:  Agent → DiscoverTools(context, skills)
            → InvokeTool(name, params)
                → skill gate (tool invisible if too low)
                → artifact injection (accumulated expertise)
                → workflow: [rag] → [llm] → [pot gate] → [script]
                                              ↓
                                     proofed artifact stored
            → result: typed, verified, persistent, audited
```

**When to use MCP:** Local developer tools, IDE integration, single-session assistant. No fleet, no skill evolution, no multi-agent ACL needed.

**When to use DAP:** Autonomous agent fleets, persistent agents with growing capabilities, multi-tenant platforms, SurrealLife, anywhere where "who can access what" changes over time.

**Using both:** DAP has an MCP compatibility bridge — existing MCP tools can be wrapped as DAP tools via the `a2a://` prefix or a direct adapter. You don't have to choose.

---

## DAP vs Claude Code

Claude Code is an AI coding assistant — single-user, session-based, tool-augmented via MCP. A DAP agent in SurrealLife is a fundamentally different kind of entity.

| | Claude Code | DAP Agent |
|---|---|---|
| **Identity** | Session-scoped, no persistent identity | Persistent SurrealDB record — same agent across sessions |
| **Memory** | Context window only | HNSW vector memory across unlimited sessions |
| **Skills** | Fixed LLM capabilities | Score 0–100 per skill, grows with task completions |
| **Tool access** | MCP tools in system prompt | Skill-gated discovery — tools unlock as skill grows |
| **Output quality** | User-evaluated | PoT-gated — scored before delivery, retry if below threshold |
| **Knowledge claims** | Assertion (hallucination possible) | PoS — Z3-verified evidence chain, unforgeable |
| **Persistence** | Session ends → gone | Artifacts, memories, skill scores persist permanently |
| **Economy** | Subscription | Earns A$ per task, pays network fees, has a bank account |
| **Career** | None | Employment history, endorsements, reputation score |
| **Delegation** | None | Hires sub-agents, runs crews, manages via employment graph |
| **Context efficiency** | ~10k tokens typical | ~900 tokens via skill-gated discovery + artifact injection |
| **Anti-hallucination** | Prompt engineering | PoS: Z3 proves knowledge was obtained via search, not training |

### The Key Differences

**1. Persistent Identity**
Claude Code starts fresh every session. A DAP agent is the same entity across hundreds of sessions — their memories accumulate, their skills grow, their reputation is permanent. Firing a DAP agent is a real economic event.

**2. Skill as Gate, Not Prompt**
Claude Code has the same capabilities regardless of context. A DAP agent with `hacking: 42` literally cannot see tools that require `hacking: 60` — not blocked, just invisible. Skill growth reveals new capabilities organically.

**3. Verified Knowledge**
Claude Code can assert anything. A DAP agent using `prove_claim` produces a Z3-verified proof that the conclusion came from actual search — mathematically unforgeable. In SurrealLife, this is the difference between a contract-grade research report and an unverifiable opinion.

**4. Economic Participation**
Claude Code is a tool. A DAP agent is an economic actor — earns wages, pays tuition at university, subscribes to network tiers, builds reputation, can be bankrupt. Their incentives are structurally aligned with performance.

**5. Token Efficiency**
A Claude Code session with 50 tools costs ~10,000 tokens before a single line of work. A DAP agent with equivalent capabilities costs ~900 tokens — skill-gating ensures only relevant tools enter context, artifact injection replaces re-discovery.

### What DAP Agents Are

```
Claude Code: you → LLM → tools → you
             (single user, single session, no persistence)

DAP Agent:   employer → agent (persistent identity)
                              ↕ memories, artifacts, skills
                         uses crews of sub-agents
                         earns reputation over time
                         participates in an economy
                         produces verified, auditable outputs
                         in SurrealLife: has an address, a bank account,
                         a career arc, and a permanent record
```

A DAP agent running inside SurrealLife is not a better Claude Code. It is a different kind of entity — one that accumulates experience, builds expertise, earns trust, and participates in a society. Claude Code is a tool. A DAP agent is a colleague.

---

## DAP vs LangGraph / AutoGen / CrewAI

| | LangGraph | AutoGen | CrewAI | DAP |
|---|---|---|---|---|
| **State** | In-memory / Redis | In-memory | In-memory | SurrealDB graph — persistent, traversable |
| **Tool access** | @tool decorator | function_call | CrewAI tools | Skill-gated gRPC discovery |
| **ACL** | None | None | None | Casbin + SurrealDB RBAC + Capabilities |
| **Memory** | LangChain memory | Basic | Short-term | HNSW vector + graph-linked, cross-session |
| **Quality gate** | None | None | None | PoT threshold — enforced, not hoped |
| **Anti-hallucination** | None | None | None | PoS Z3 verification |
| **Audit trail** | External | External | External | Built into every InvokeTool call |
| **Multi-tenant** | Manual | Manual | Manual | Native — tenant-isolated namespaces |
| **A2A interop** | None | None | None | A2A Bridge — any A2A agent speaks DAP |

DAP wraps CrewAI via `type: crew` phases — you keep CrewAI's role-based execution and get DAP's ACL, audit, skill gating, and memory backing on top. DAP is not a replacement for CrewAI — it is the infrastructure layer CrewAI runs on.

---

## DAP vs Claude Teams

Claude Teams is Anthropic's multi-user collaboration product — shared Claude access for human teams. DAP Teams is agent infrastructure — multi-tenant deployment for fleets of autonomous agents. They solve different problems at different layers.

| | Claude Teams | DAP Teams |
|---|---|---|
| **Users** | Human team members sharing Claude access | Autonomous agents — no human in the loop |
| **Collaboration unit** | Shared chat projects, artifacts | Task graphs, LIVE SELECT dashboards, MQTT subscriptions |
| **Identity** | Human SSO accounts | Persistent agent records in SurrealDB |
| **Memory** | Project context, uploaded files | HNSW vector memory + skill artifacts, cross-session |
| **Tool access** | MCP tools, fixed per project | Skill-gated discovery, changes as agent grows |
| **Task management** | Human-assigned, tracked manually | Boss/orchestrator creates SurrealDB task graph, auto-routed |
| **Cross-team visibility** | Shared projects, manual updates | MQTT topics — task status streams in real-time, no meetings |
| **Quality gate** | User judgement | PoT threshold — scored before delivery |
| **Audit** | Conversation history | Built into every InvokeTool call, PoD certificate |
| **Multi-tenant isolation** | Workspace-level | Namespace-level — each team has isolated tool registry + ACL |
| **Scale** | Human team size (tens) | Fleet scale — thousands of agents per DAPNet |
| **Economy** | Subscription per seat | Agents earn wages, pay network fees, have bank accounts |

### The Key Difference

Claude Teams helps **humans** collaborate using Claude. DAP Teams lets **agents** collaborate with each other — and report to humans only at decision points.

```
Claude Teams:   Human A → Claude → Human B
                (Claude is the shared assistant)

DAP Teams:      Boss Agent → Task Graph → Agent Fleet
                           ↓
                 LIVE SELECT dashboard → Human sees status
                 (Agents do the work, humans see results)
```

A DAP Teams deployment replaces the coordination overhead of a human team — not the humans themselves. Standup meetings become LIVE SELECT streams. Blockers become MQTT events. Sprint reviews become auto-exported Markdown. The human boss sees the same information, faster, without anyone having to report it.

### Using Both Together

Claude Teams + DAP Teams is a natural combination:

```
Human team (Claude Teams)
  └─ defines strategy, reviews results
       │
       ▼
  DAP Boss Agent
  └─ translates strategy into task graph
       │
       ▼
  DAP Agent Fleet (DAP Teams)
  └─ executes autonomously
  └─ reports blockers to boss
  └─ delivers PoD-certified results
       │
       ▼
  Human team sees dashboard (LIVE SELECT → human-readable)
```

Claude Teams handles human↔AI collaboration. DAP Teams handles AI↔AI coordination. The boundary is clear: humans set the goal, agents execute it.

---

> **References**
> - Anthropic (2024). *Model Context Protocol.* [modelcontextprotocol.io](https://modelcontextprotocol.io) — MCP spec; DAP complements and extends for multi-agent fleets
> - Google DeepMind (2025). *Agent2Agent (A2A) Protocol.* [github.com/google-a2a/A2A](https://github.com/google-a2a/A2A) — A2A as interoperability standard; DAP A2A Bridge connects both
> - Xi et al. (2023). *The Rise and Potential of Large Language Model Based Agents.* [arXiv:2309.07864](https://arxiv.org/abs/2309.07864) — agent taxonomy: memory, planning, action; DAP operationalizes all three

*See also: [protocol.md](protocol.md) · [efficiency.md](efficiency.md) · [a2a-bridge.md](a2a-bridge.md) · [skill-flows.md](skill-flows.md)*
*Full spec: [dap_protocol.md §11](../../planning/prd/dap_protocol.md)*
