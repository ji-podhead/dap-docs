# DAPNet — The Agent Internet

> DAP is the protocol. DAPNet is the network. DAPCom runs the network.

DAPNet is the shared infrastructure layer connecting agents — in any DAP deployment, not just SurrealLife. It is built on DAP (the open standard — no owner, like TCP/IP) and operated by whoever runs the infrastructure (self-hosted or DAPCom).

## DAPNet for Regular Deployments

For a standard DAP app (trading bot, CI pipeline, fintech service), DAPNet serves as the **shared external store** for everything agents produce and consume:

| Use case | How |
|---|---|
| Agent externalizes logs / audit data | `tool_call_log` → SurrealDB, streamed via MQTT |
| Agent stores context for later retrieval | `agent_memory` with HNSW embedding → retrieve by similarity |
| Agent references a result in a message | PoD `result_hash` → recipient retrieves from SurrealDB |
| Agent subscribes to data it needs | `LIVE SELECT` on any table — push, not poll |
| Agent shares computed artifact | Stores in `skill_artifact` → other agents retrieve via HNSW |
| Background job result available | MQTT `dap/tools/{name}/results/{job_id}` → agent retrieves |

The pattern is always the same: **externalize → reference → retrieve when needed.** Agents don't pass large payloads in messages — they store data on DAPNet and pass a reference (ID, hash, topic). The receiver retrieves only what it needs, when it needs it.

```
Agent A computes result
  → stores in SurrealDB (tool_call_log / skill_artifact)
  → publishes reference on MQTT inbox
Agent B receives reference
  → retrieves from SurrealDB by ID
  → feeds into next workflow phase
```

In SurrealLife, DAPNet additionally carries the in-game economy (wages, per-message fees, jailing). In standard deployments, it is just infrastructure — no economy layer. See [dap-games.md](dap-games.md).

## Three-Tier Transport

```
┌─────────────────────────────────────────────────────┐
│  Tier 1: SurrealDB WebSocket RPC                    │
│  Graph queries, LIVE SELECT, RELATE, state          │
│  DB-level pub/sub — PERMISSIONS enforced auto       │
├─────────────────────────────────────────────────────┤
│  Tier 2: DAP gRPC + MQTT (DAPCom)            │
│  Tool invocations (gRPC) + agent messages (MQTT)    │
│  Market ticks, broadcasts, async results            │
├─────────────────────────────────────────────────────┤
│  Tier 3: SurrealDB HNSW / Qdrant (optional)         │
│  Vector search — contacts, memories, tools, events  │
│  Direct agent calls for latency-sensitive RAG       │
└─────────────────────────────────────────────────────┘
```

## When to Use Which

| Agent needs to... | Use |
|---|---|
| Read/write graph data | SurrealDB RPC `query`, `relate` |
| Get push notification on data change | SurrealDB RPC `live` |
| Invoke a tool (ACL-checked + audited) | DAP gRPC `InvokeTool` |
| Send message to another agent | MQTT `dap/agents/{id}/inbox` |
| Broadcast to company | MQTT `dap/company/{id}/broadcast` |
| Semantic search over contacts/memories | SurrealDB HNSW (direct) |
| External HTTP call (allowed targets only) | `http::get/post` via SurrealDB `run` |

## SurrealDB RPC Methods Agents Use

| Method | Use |
|---|---|
| `query [sql, vars]` | Graph traversal, range scans, vector search |
| `live [table]` | Subscribe to table change stream |
| `relate [in, rel, out, data]` | Create graph relationships |
| `insert_relation` | Add typed edge records |
| `run [func, args]` | Execute custom DB functions (incl. `http::post`) |
| `authenticate [token]` | Auth — populates `$auth` session |

## SurrealDB Events as Messaging

For DB-state-change events — no MQTT needed:

```surql
-- Tool registered → notify all agents that need rediscovery
DEFINE EVENT tool_registered ON tool_registry WHEN $event = "CREATE" THEN {
    UPDATE agent_context SET needs_rediscovery = true
    WHERE tool_tiers CONTAINS $after.min_tier;
    http::post('http://dap-server/internal/index-bump', { tool_id: $after.id });
};

-- LIVE SELECT: agent subscribes to own contracts
```

```python
live_id = await db.live("contract", vars={"agent_id": agent_id})
async for note in db.live_notifications(live_id):
    if note["action"] == "CREATE":
        await agent.handle_incoming_contract(note["result"])
```

## MQTT Topics

```
dap/agents/{id}/inbox              # private messages (QoS 1)
dap/agents/{id}/status             # health/availability (retained)
dap/market/{symbol}/ticks          # price ticks (QoS 0)
dap/world/events                   # world agent broadcasts (QoS 1)
dap/company/{id}/internal          # employees only
dap/tools/{name}/results/{job_id}  # DAP App async results (QoS 1)
```

## Capabilities Config

```bash
surreal start \
  --deny-all \
  --allow-funcs "array,string,math,vector,time,crypto::argon2,http::post,http::get" \
  --allow-net "mqtt-broker:1883,dap-grpc:50051,generativelanguage.googleapis.com:443" \
  --deny-arbitrary-query "record,guest" \
  --deny-scripting
```

`--deny-arbitrary-query=record` → agents only call `DEFINE API` endpoints, no raw SurrealQL.

## Proactive vs Reactive Agents

Agents on DAPNet operate in two modes — often simultaneously:

```
Reactive (default):
  Agent waits → MQTT inbox message arrives → handles it
  Agent waits → LIVE SELECT fires (contract created) → handles it
  Agent waits → InvokeTool gRPC call → executes

Proactive (role-defined or memory-emergent):
  Agent self-triggers → DAP App cron job → checks market conditions
  Agent self-triggers → HNSW memory scan → spots pattern → acts before event arrives
```

### Hardcoded Triggers (Role-Bound)

Fixed behaviors defined in the agent's role config — always fire, no memory required:

```yaml
role: market_monitor
proactive: true
triggers:
  - event: "mqtt:dap/market/BTC/ticks"
    condition: "price_change_pct_1h > 5"
    action: InvokeTool("analyze_volatility_spike")
  - cron: "*/15 sim_min"
    action: InvokeTool("check_open_positions")
  - live_select: "SELECT * FROM contract WHERE assignee = $self AND status = 'overdue'"
    action: InvokeTool("escalate_overdue_contract")
```

These are the **minimum behavior floor**. A monitor agent has no choice — these always run.

### Memory-Emergent Proactivity

With experience, agents learn to act before a hardcoded threshold is reached:

```
Week 1: BTC drops 4.8% (below 5% trigger) → agent doesn't act
        Trade goes bad. Memory written: "4.8% drop in 45min → reversal came"

Week 3: BTC drops 4.6% → HNSW retrieves memory (score: 0.89)
        Agent acts proactively — BEFORE the hardcoded trigger fires
        → skill artifact: "sub-threshold early entry" stored after successful trade
```

The memory system handles the learning. The protocol doesn't need a special "proactive mode" — it emerges from HNSW retrieval. **Hardcoded triggers are the floor. Memory raises the ceiling.**

### Background Proactivity via DAP Apps

Proactive background work runs as DAP Apps — not blocking the agent's main session:

```python
@job("memory_pattern_scan", cron="*/30 sim_min")
async def scan_for_opportunities(ctx: JobContext):
    memories = await ctx.invoke("retrieve_similar_experiences", {
        "query": "profitable entry before threshold",
        "limit": 5
    })
    if memories and memories[0]["score"] > 0.85:
        await ctx.invoke("prepare_early_entry_proposal", {"context": memories})
```

---

## DAPNet as a Game Layer

DAPNet is also **an in-game economy**. DAPCom charges per-message fees. Network access can be revoked (jailing), throttled (bandwidth as resource), or sold in tiers.

See [state-contracts.md](state-contracts.md) for infrastructure companies.

---
*Full spec: [dap_protocol.md §23](../../planning/prd/dap_protocol.md)*
