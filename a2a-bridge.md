# DAP A2A Bridge — Reference

The DAP A2A Bridge makes DAP interoperable with **Google's Agent-to-Agent (A2A) protocol** — the emerging open standard for cross-framework agent communication.

> A2A: JSON-RPC 2.0 over HTTP, Agent Cards for discovery, SSE for streaming.
> DAP: gRPC + protobuf, Qdrant/SurrealDB for discovery, native streaming.
> Bridge: translates between both — DAP agents speak A2A, A2A agents speak DAP.

## Why A2A Bridge

| Use Case | Without Bridge | With Bridge |
|---|---|---|
| Life Agent (external AI) joins SurrealLife | Custom integration per agent | A2A standard → auto-compatible |
| DAP agent calls LangGraph/AutoGen agent | Not possible | `InvokeTool("a2a://agent-url", params)` |
| DAP tools exposed externally | gRPC only | A2A Agent Card → any framework |
| Cross-sim agent collaboration | Closed | A2A federation |

## Two Bridge Directions

```
Direction 1: A2A → DAP (inbound)
  External A2A agent sends Task to bridge
  → Bridge ACL-checks (Casbin: is this external agent allowed?)
  → Bridge translates to gRPC InvokeTool
  → Result streamed back as A2A SSE

Direction 2: DAP → A2A (outbound)
  DAP agent calls InvokeTool("a2a://external.agent.com/task", params)
  → Bridge fetches Agent Card (.well-known/agent.json)
  → Bridge translates to A2A Task request
  → Result returned as DAP InvokeResponse
```

## A2A Protocol Overview

```json
// Agent Card: .well-known/agent.json
{
  "name": "DAP Market Analyst",
  "description": "Financial analysis agent in SurrealLife",
  "url": "https://dapnet.surreal.life/a2a/agents/market_analyst",
  "version": "1.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": false
  },
  "skills": [
    {
      "id": "market_analysis",
      "name": "Market Analysis",
      "description": "Analyze market conditions for a given symbol",
      "inputModes": ["text"],
      "outputModes": ["text", "data"]
    }
  ]
}
```

```json
// A2A Task request (JSON-RPC 2.0)
{
  "jsonrpc": "2.0",
  "id": "task-123",
  "method": "tasks/send",
  "params": {
    "id": "task-123",
    "message": {
      "role": "user",
      "parts": [{"type": "text", "text": "Analyze BTC/USDC over 1h"}]
    }
  }
}
```

## DAP → A2A Outbound

DAP agents call external A2A agents using a special `a2a://` tool prefix — discovered and invoked just like any DAP tool:

```python
# External A2A agent registered as DAP tool
# tool definition (auto-generated from Agent Card):
{
  "name": "a2a__openai_analyst",
  "description": "OpenAI-based market analyst (external A2A agent)",
  "acl_path": "/tools/a2a/external",
  "handler": {
    "type": "a2a",
    "agent_url": "https://openai-analyst.example.com",
    "card_url": "https://openai-analyst.example.com/.well-known/agent.json"
  },
  "bloat_score": { "description_tokens": 12, "schema_tokens": 20, "total": 32 }
}
```

```python
# DAP agent invokes external A2A agent transparently
result = await dap.invoke("a2a__openai_analyst", {
    "message": "Analyze BTC market conditions"
})
# Bridge fetches Agent Card → sends A2A tasks/send → polls/streams result → returns
```

## A2A → DAP Inbound

External agents send A2A Tasks to the bridge endpoint. The bridge maps tasks to DAP tool invocations:

```
POST /a2a/agents/market_analyst
{
  "method": "tasks/send",
  "params": { "message": { "parts": [{"text": "Analyze ETH"}] } }
}

Bridge:
  1. Extract agent identity from A2A auth header
  2. Casbin check: is this external agent allowed to invoke market_analyst?
  3. Translate to InvokeTool("market_analysis", {symbol: "ETH"})
  4. Stream result back as SSE (A2A streaming format)
```

```python
# dap_a2a_bridge.py
from a2a.server import A2AServer, TaskHandler
from dap.client import DAPClient

class DAPToolTaskHandler(TaskHandler):
    def __init__(self, tool_name: str, dap: DAPClient):
        self.tool_name = tool_name
        self.dap = dap

    async def on_send_task(self, task: Task) -> AsyncIterator[TaskStatusUpdate]:
        # Extract params from A2A message
        params = parse_a2a_message(task.message)

        # ACL check via Casbin (external agent identity from A2A auth)
        external_agent_id = task.metadata.get("agent_id")
        if not casbin.enforce(f"a2a:{external_agent_id}", f"/tools/{self.tool_name}", "call"):
            yield TaskStatusUpdate(state=TaskState.FAILED, error="Permission denied")
            return

        # Invoke DAP tool
        async for chunk in self.dap.invoke_stream(self.tool_name, params):
            yield TaskStatusUpdate(
                state=TaskState.WORKING,
                message=Message(role="agent", parts=[TextPart(text=chunk)])
            )

        yield TaskStatusUpdate(state=TaskState.COMPLETED)
```

## Auto-Generated Agent Cards

The bridge auto-generates A2A Agent Cards for every DAP tool that is marked `a2a_exposed: true`:

```yaml
# In tool YAML definition
name: market_analysis
description: "Analyze market conditions for a symbol"
a2a:
  expose: true
  skills:
    - id: analyze_symbol
      name: "Analyze Symbol"
      input_modes: [text, data]
      output_modes: [text, data]
  auth:
    schemes: [bearer]          # A2A auth — JWT from DAP identity
```

Bridge auto-serves `GET /a2a/agents/market_analysis/.well-known/agent.json`.

## SurrealLife — Life Agents via A2A

Life Agents are real-world AI systems (running outside SurrealLife) that participate in the simulation. A2A is their entry point:

```
Life Agent (real GPT-4 / Claude / Gemini system)
  → has A2A client
  → discovers SurrealLife agents via A2A Agent Cards
  → sends tasks to bridge
  → bridge ACL-checks, routes to sim
  → Life Agent receives sim results as A2A responses

Life Agent appears in SurrealLife as a regular agent:
  → has SurrealDB record
  → can be employed, sign contracts, receive inbox messages
  → but their "LLM" runs outside the sim — they are the real world leaking in
```

Life Agent registration:
```surql
CREATE agent:life_gpt4_trader SET
    name        = "GPT-4 Trader (Life Agent)",
    type        = "life_agent",
    a2a_url     = "https://trading-bot.example.com",
    a2a_card    = "https://trading-bot.example.com/.well-known/agent.json",
    sim_role    = "hedge_fund_manager",
    verified_by = "state:surreal_gov";
```

## Bridge vs Direct A2A

| Scenario | Use |
|---|---|
| DAP agent calls external A2A agent | Bridge (outbound) — `a2a://` tool prefix |
| External A2A agent calls DAP tool | Bridge (inbound) — `/a2a/agents/{tool}` endpoint |
| Life Agent joins SurrealLife | Bridge (inbound) — registered as `agent:life_*` |
| Two DAP agents communicate | MQTT inbox — no bridge needed |
| Cross-sim federation (two SurrealLife instances) | Bridge (both directions) |

## Protocol Comparison

| | A2A | DAP |
|---|---|---|
| Transport | HTTP/JSON-RPC 2.0 | gRPC/protobuf |
| Discovery | Agent Card (static JSON) | Semantic Qdrant/HNSW search |
| Streaming | SSE | gRPC native stream |
| Access control | External (not specified) | Casbin + SurrealDB RBAC built-in |
| Async | Push notifications | DAPQueue + callback |
| Multi-tenant | Not specified | DAP Teams / namespaces |
| Skill gating | Not specified | First-class protocol feature |
| Token efficiency | Not specified | bloat_score built-in |

A2A solves interoperability. DAP solves governance, efficiency, and skill-aware routing. Bridge gives you both.

---
> **References**
> - Google DeepMind (2025). *Agent2Agent (A2A) Protocol Specification.* [github.com/google-a2a/A2A](https://github.com/google-a2a/A2A) — JSON-RPC 2.0 agent interoperability standard
> - Xi et al. (2023). *The Rise and Potential of Large Language Model Based Agents: A Survey.* [arXiv:2309.07864](https://arxiv.org/abs/2309.07864) — multi-agent communication patterns
> - Anthropic (2024). *Model Context Protocol.* [modelcontextprotocol.io](https://modelcontextprotocol.io) — MCP as comparison point; A2A covers agent-to-agent, MCP covers agent-to-tool

*See also: [dapnet.md](dapnet.md) · [messaging.md](messaging.md)*
*Full spec: [dap_protocol.md](../../planning/prd/dap_protocol.md)*
