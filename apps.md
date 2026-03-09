# DAP Apps — Reference

DAP Apps extends DAP with an **async message-queue invocation model**. Not every tool call should be a blocking gRPC connection. Long-running tools, fan-out to sub-agents, and sim-phase workflows flow through DAPQueue — the agent publishes, gets a `job_id` immediately, and receives the result via callback when ready.

> Inspired by Slack Bolt / Cloudflare Queue Workers — but for agent tool calls.

## When to Use Async Instead of Sync gRPC

| Use `sync` gRPC | Use `async` DAPQueue |
|---|---|
| Fast tools (<5s) | Long-running workflows (hours) |
| Interactive responses | Background analysis |
| Single result | Fan-out to multiple workers |
| Agent holds connection | Agent crashes → resume from job_id |
| Simple tool | SimEngine phases (sim-time pauses) |

## Four Invocation Modes

| Mode | How | Returns |
|---|---|---|
| `sync` | gRPC `InvokeTool` blocking | Result directly |
| `stream` | gRPC `InvokeTool` streaming | Progress chunks |
| `async` | Queue publish | `job_id` immediately |
| `broadcast` | Queue fan-out → N workers | `[job_id, ...]` |

```python
# Sync
result = await dap.invoke("web_search", {"query": "BTC market cap"})

# Async — agent continues other work
job_id = await dap.invoke_async("full_market_analysis", {"symbols": ["BTC","ETH"]})
result = await dap.poll(job_id, timeout=sim_hours(4))

# Broadcast — parallel dispatch
job_ids = await dap.broadcast("analyze_sector", sectors, workers=4)
results = await dap.gather(job_ids)
```

## DAP App Tool Definition

```yaml
name: full_market_analysis
skill_required: data_analysis
skill_min: 45

app:
  execution_mode: async
  max_runtime_sim_hours: 8
  concurrency: 1               # max 1 concurrent per agent
  retry:
    max_attempts: 3
    backoff: exponential
    dead_letter: true          # failed jobs → DLQ, agent notified
  callback:
    mode: redis_channel        # result → {agent_id}:dap:results
    fallback: poll

handler:
  type: workflow
  ref: workflows/full_market_analysis.yaml.j2
```

## Worker Pool

```python
from dap.worker import DAPWorker, job

worker = DAPWorker(
    queue="redis://localhost:6379",
    server="grpc://localhost:50051",
    namespace="market_tools",
)

@job("full_market_analysis")
async def handle_analysis(params: dict, ctx: JobContext):
    for symbol in params["symbols"]:
        data = await ctx.invoke("fetch_ohlcv", {"symbol": symbol})
        ctx.emit_progress(f"Fetched {symbol}")    # streams to agent if subscribed
    return await ctx.invoke("run_correlation", {"data": data})

worker.run()
```

`ctx.invoke` re-enters DAP gRPC — ACL-checked and audited. Workers are stateless.

## Architecture

```
Agent → DAPQueue (Redis/NATS/Kafka)
            ↓
       Worker Pool
         ACL check → skill check → execute → publish result
            ↓
       Result Store (SurrealDB / Redis)
         job_id → {status, result, error, ttl}
            ↓
       Agent callback (Redis channel) or poll
```

## SurrealLife — SimEngine Phases as Queue Checkpoints

In SurrealLife workflows, `simengine` phases become queue checkpoints. The agent's connection stays closed — the job resumes when the sim-clock advances:

```
Worker: Phase 1 llm → result stored
Worker: publishes sim_wait → SimEngine
SimEngine: advances clock, generates counter-events
Worker: resumes Phase 2 with counter-event context
Agent: receives final result via callback channel
```

Outside SurrealLife, `simengine` phases don't exist — DAP Apps work identically for `llm`, `script`, `crew` phases.

## Backends

| Backend | Best for |
|---|---|
| Redis Streams | Default, same infra as existing Redis |
| NATS JetStream | Ultra-low latency |
| Kafka | High throughput, audit-grade retention |
| MQTT | SurrealLife fan-out to many agents |

---
*Full spec: [dap_protocol.md §21](../../planning/prd/dap_protocol.md)*
