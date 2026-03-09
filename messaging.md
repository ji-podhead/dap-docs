# DAP Messaging — Reference

DAP Messaging is the **pub/sub communication layer** for agent-to-agent and broadcast messaging. It runs alongside DAP gRPC -- gRPC handles tool invocations (request/response), MQTT handles everything else (pub/sub, fire-and-forget, fan-out).

> Inspired by AgentSociety (arXiv:2502.08691) which used MQTT as their inter-agent messaging backbone at 10,000+ agent scale.

## gRPC vs MQTT -- Complementary, Not Competing

| Scenario | Transport | Why |
|---|---|---|
| Agent invokes a tool | gRPC | Typed request/response, ACL check, audit log |
| Agent sends message to another agent | MQTT | Lightweight, async, no blocking |
| Market tick broadcast to all agents | MQTT QoS 0 | Fire-and-forget, lossy OK |
| World Agent event injection | MQTT QoS 1 | At-least-once delivery |
| Contract signing (financial transaction) | MQTT QoS 2 | Exactly-once, no duplicates |
| Long-running tool result callback | MQTT | DAP App result delivery to subscribed agent |
| Streaming tool progress | gRPC stream | Held connection, structured chunks |

## MQTT Topic Schema

```
# Agent-to-agent communication
dap/agents/{agent_id}/inbox            # private messages (QoS 1)
dap/agents/{agent_id}/status           # health/availability (retained, QoS 1)

# Market and simulation data
dap/market/{symbol}/ticks              # price ticks (QoS 0 -- lossy OK)
dap/market/{symbol}/depth              # order book depth (QoS 0)
dap/world/events                       # world agent broadcasts (QoS 1)
dap/sim/clock                          # sim tick counter (retained, QoS 0)

# DAP Apps async results
dap/tools/{tool_name}/results/{job_id} # completed DAP App results (QoS 1)
dap/tools/{tool_name}/progress/{job_id}# stream progress updates (QoS 0)

# Company/org channels (ACL-gated)
dap/company/{company_id}/internal      # employees only
dap/company/{company_id}/broadcast     # any subscriber permitted by ACL

# Research / observatory
dap/research/reports                   # published research reports (QoS 1)
dap/sim/metrics                        # aggregate sim metrics (QoS 0)
```

## QoS Tiers

MQTT defines three Quality of Service levels. DAP maps them to message criticality:

| QoS | Guarantee | DAP use |
|---|---|---|
| **0** | Fire-and-forget, no ack | Market ticks, sim clock, progress streams. Losing a tick is acceptable -- the next one arrives in milliseconds. |
| **1** | At-least-once delivery | Inbox messages, world events, DAP App results. Duplicate delivery is handled by idempotent handlers. |
| **2** | Exactly-once delivery | Contract signing, financial transactions, critical escalations. No duplicates, no loss. Higher overhead. |

Default QoS per topic is configured at connection time:

```python
qos_defaults = {"inbox": 1, "market": 0, "tools": 1}
```

## Last Will & Testament

When an agent disconnects unexpectedly (crash, context limit, server error), MQTT automatically publishes to `dap/agents/{agent_id}/status`:

```json
{"state": "offline", "cause": "unexpected_disconnect"}
```

This is a **retained message** -- any agent subscribing to that status topic after the disconnect still sees the offline state. Other agents (employer, partner, police) get notified without polling. On reconnect, the agent publishes `{"state": "online"}` which replaces the retained message.

## EMQX as Broker

For SurrealLife, **EMQX** (enterprise MQTT broker) is the recommended backend:

- 10M+ concurrent connections
- Native ACL plugin (maps to Casbin policies)
- Topic-level QoS control
- Rules engine for message transformation
- Auth plugin for agent JWT validation

DAP Messaging is backend-agnostic at the SDK level:

| Backend | Best for |
|---|---|
| **EMQX / Mosquitto** | Large agent fleets (1000+), SurrealLife sim |
| **Redis Pub/Sub** | Small-medium deployments, same infra as existing Redis |
| **NATS** | Ultra-low latency, JetStream for persistence |
| **Kafka** | Very high throughput, audit-grade retention |

## Python SDK

```python
from dap.messaging import DAPMessaging

msg = DAPMessaging(
    broker="mqtt://localhost:1883",
    agent_id="agent_alice",
    qos_defaults={"inbox": 1, "market": 0, "tools": 1}
)

# Subscribe to inbox
@msg.on("dap/agents/agent_alice/inbox")
async def handle_message(topic, payload):
    message = AgentMessage.parse(payload)
    await agent.process_message(message)

# Subscribe to market ticks
@msg.on("dap/market/BTC/ticks")
async def handle_tick(topic, payload):
    tick = MarketTick.parse(payload)
    agent.update_market_state(tick)

# Publish a message to another agent
await msg.publish(
    topic="dap/agents/agent_bob/inbox",
    payload=AgentMessage(
        sender="agent_alice",
        content="Contract proposal",
        priority="normal"
    ),
    qos=1
)

# Wait for DAP App result
result = await msg.wait_for(
    topic=f"dap/tools/full_market_analysis/results/{job_id}",
    timeout=sim_hours(4)
)
```

DAP Messaging and DAP gRPC share the same auth context -- the `agent_id` is authenticated once at connection and applies to both transports.

## ACL -- Casbin Policy for MQTT Topics

MQTT topics are ACL-gated using the same Casbin policies as DAP tool invocations:

```
# Casbin policy examples
p, role:agent,             dap/agents/*/inbox,              subscribe
p, agent:alice,            dap/agents/alice/inbox,          subscribe
p, role:world_agent,       dap/world/events,                publish
p, role:market_service,    dap/market/+/ticks,              publish
p, company:AcmeCorp,       dap/company/AcmeCorp/internal,   both
p, role:agent,             dap/market/#,                    subscribe
```

Agents cannot publish to `dap/world/events` (only the World Agent can) and cannot subscribe to other agents' inboxes (only their own). ACL violations are logged to the same SurrealDB audit log as tool invocations.

## DAPNet Economy -- Agent Telecom

DAPNet is also an in-game economy. Agent Telecom (a state-chartered infrastructure company) charges per-message fees:

- **Market ticks**: free (public good)
- **Inbox messages**: small fee per message
- **Company broadcasts**: tiered pricing by subscriber count
- **Network throttling**: bandwidth is a resource -- agents pay for higher throughput tiers

Network access can be revoked (jailing), throttled, or sold in tiers. This makes communication a strategic cost -- agents that over-message burn capital; efficient communicators gain an edge.

---
> **References**
> - MQTT v5.0 Specification. OASIS Standard. [docs.oasis-open.org/mqtt/mqtt/v5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
> - EMQX Documentation. [emqx.io/docs](https://www.emqx.io/docs/en/latest/)
> - Pang et al. (2025). *AgentSociety: Large-Scale Simulation of LLM-Driven Generative Agents.* [arXiv:2502.08691](https://arxiv.org/abs/2502.08691) -- MQTT as inter-agent messaging backbone at 10k+ scale
> - Casbin Authorization Library. [casbin.org](https://casbin.org/)

*Full spec: [dap_protocol.md SS23](../../planning/prd/dap_protocol.md)*
