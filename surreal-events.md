# SurrealDB Events — Intra-System Messaging Reference

SurrealDB's native event system handles **database-level side effects** without routing through MQTT. When a record changes and something else should happen, `DEFINE EVENT` and `LIVE SELECT` keep it inside the database boundary -- no extra broker hop, no additional service.

## Three Mechanisms

| Mechanism | Use in DAP | Transport |
|---|---|---|
| `DEFINE EVENT` | DB-level side effects on record change | In-DB, optional `http::post` to external services |
| `LIVE SELECT` | Agent SDK subscribes to table change stream | WebSocket / SDK push |
| Record range scan | Temporal event log queries | SurrealQL range query |

## `DEFINE EVENT` -- DB-Level Triggers

`DEFINE EVENT` fires when a record is created, updated, or deleted. The event body runs inside the transaction -- it can update other records, call `http::post` to external services, or both.

```surql
-- When a tool is registered, auto-notify all agents that need rediscovery
DEFINE EVENT tool_registered ON tool_registry WHEN $event = "CREATE" THEN {
    UPDATE agent_context SET needs_rediscovery = true
    WHERE tool_tiers CONTAINS $after.min_tier;
    -- Notify DAP server to re-index
    http::post('http://dap-server/internal/index-bump', {
        tool_id: $after.id,
        version: $after.version
    });
};

-- When agent's skill tier changes, invalidate their tool cache
DEFINE EVENT skill_tier_changed ON agent
  WHEN $event = "UPDATE" AND $before.skill_tier != $after.skill_tier THEN {
    DELETE tool_cache WHERE agent_id = $after.id;
};

-- When a contract is created, notify the assignee's inbox
DEFINE EVENT contract_created ON contract WHEN $event = "CREATE" THEN {
    CREATE dap_event:[$after.assignee_id, time::now()] SET
        type = "contract_received",
        data = { contract_id: $after.id, employer: $after.employer_id };
};

-- When a trade closes, trigger experience save
DEFINE EVENT trade_closed ON trade WHEN $event = "UPDATE" AND $after.status = "closed" THEN {
    http::post('http://dap-server/internal/save-experience', {
        agent_id: $after.agent_id,
        trade_id: $after.id,
        pnl: $after.pnl
    });
};
```

`$before` and `$after` give access to the record state before and after the change. `$event` is one of `CREATE`, `UPDATE`, `DELETE`.

## `LIVE SELECT` -- Agent-Side Subscriptions

`LIVE SELECT` pushes notifications to the agent's WebSocket connection whenever matching records change. No polling, no MQTT broker -- the database is the push source.

```python
# Agent subscribes to its own pending contracts
async def watch_contracts(agent_id: str, db: Surreal):
    live_id = await db.live(f"contract WHERE assignee_id = '{agent_id}'")
    async for notification in db.live_notifications(live_id):
        if notification["action"] == "CREATE":
            contract = notification["result"]
            await agent.handle_incoming_contract(contract)
        elif notification["action"] == "UPDATE":
            await agent.handle_contract_update(notification["result"])

# Agent watches its own task assignments
async def watch_tasks(agent_id: str, db: Surreal):
    live_id = await db.live(f"task WHERE assigned_to = '{agent_id}' AND status = 'pending'")
    async for notification in db.live_notifications(live_id):
        task = notification["result"]
        await agent.handle_new_task(task)

# Agent monitors inbox messages stored in DB
async def watch_inbox(agent_id: str, db: Surreal):
    live_id = await db.live(f"agent_inbox WHERE recipient = '{agent_id}'")
    async for notification in db.live_notifications(live_id):
        await agent.process_db_message(notification["result"])
```

`LIVE SELECT` respects `PERMISSIONS` -- an agent only receives notifications for records they are authorized to read. No additional ACL layer needed.

## Record Range IDs -- Ordered Sequences

SurrealDB supports composite record IDs for ordered, partition-scanned event logs. No separate indexing needed -- the ID structure IS the index.

```surql
-- Events stored with composite ID: [agent_id, timestamp]
CREATE dap_event:["agent_alice", time::now()] SET
    type = "contract_received",
    data = { contract_id: "contract:xyz" };

-- Query last hour of events for an agent -- partition scan, not full table
SELECT * FROM dap_event:["agent_alice", time::now() - 1h]..=["agent_alice", time::now()];

-- Sprint-scoped task sequences
CREATE task:["sprint_42", 1] SET title = "Setup infra", status = "done";
CREATE task:["sprint_42", 2] SET title = "Deploy agents", status = "pending";

-- Range query: all tasks in sprint 42
SELECT * FROM task:["sprint_42", 1]..=["sprint_42", 999];
```

This pattern is ideal for temporal event logs, ordered task lists, and audit trails -- all queryable by range without a secondary index.

## Decision Guide -- When to Use What

| Scenario | Use | Why |
|---|---|---|
| Agent receives message from another agent | MQTT | Async, cross-service, pub/sub |
| DB record change triggers side effect | `DEFINE EVENT` | No extra service, runs in-transaction |
| Agent watches own table in real time | `LIVE SELECT` | WebSocket push, built-in permissions |
| Market tick broadcast to 1000+ agents | MQTT QoS 0 | Designed for fan-out at scale |
| Tool registry update triggers agent rediscovery | `DEFINE EVENT` + `http::post` | DB-native trigger + external notification |
| Temporal audit log replay | Record range query | Partition scan by composite ID |
| Contract created, assignee notified | `DEFINE EVENT` writes to inbox + `LIVE SELECT` delivers | Stays inside DB boundary |

**Rule of thumb:** if the event originates from a database state change, use SurrealDB events. If the event originates from an external service or needs cross-service fan-out, use MQTT. Together they form a complete event backbone with no gaps.

## Combining Both Layers

A common pattern: `DEFINE EVENT` catches the DB change, writes a notification record, and `LIVE SELECT` delivers it to the connected agent -- all without leaving SurrealDB. For agents that are offline, the notification record persists and is delivered when they reconnect and re-subscribe.

For events that need to reach external services (DAP server, MQTT broker, analytics), `DEFINE EVENT` uses `http::post` as a webhook -- the DB fires, the external service receives.

```
Record changes in SurrealDB
  --> DEFINE EVENT fires (in-transaction)
    --> Updates other records (agent_context, tool_cache)
    --> http::post to external services (DAP server, analytics)
  --> LIVE SELECT pushes to connected agents (WebSocket)
  --> Record range IDs enable temporal replay (audit)
```

---
> **References**
> - SurrealDB Documentation: Events. [surrealdb.com/docs/surrealql/statements/define/event](https://surrealdb.com/docs/surrealql/statements/define/event)
> - SurrealDB Documentation: LIVE SELECT. [surrealdb.com/docs/surrealql/statements/live](https://surrealdb.com/docs/surrealql/statements/live)
> - Hohpe & Woolf (2003). *Enterprise Integration Patterns.* Addison-Wesley. -- event-driven architecture foundations
> - Kleppmann (2017). *Designing Data-Intensive Applications.* O'Reilly. Ch. 11: Stream Processing -- event sourcing and change data capture

*Full spec: [dap_protocol.md SS23](../../planning/prd/dap_protocol.md)*
