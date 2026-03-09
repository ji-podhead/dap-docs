# DAP Bench — Protocol-Level Benchmark Suite

DAP Bench is the standardized evaluation suite for DAP server implementations. It measures protocol-level behavior — discovery quality, invocation reliability, and ACL accuracy — producing a comparable server-level score published on DAP Hub.

> DAP Bench is itself a DAP artifact — a `core` package in DAP Hub. It is the instrument that generates tool and server scores.

## Three Benchmark Families

### Family A — Discovery Quality

How well does `DiscoverTools` find the right tools for a given context?

| Test | Measures | Score |
|---|---|---|
| `precision@k` | Are the top-k returned tools relevant to the task? | 0.0–1.0 |
| `recall@coverage` | What fraction of all relevant tools appear in the top-k? | 0.0–1.0 |
| `bloat_efficiency` | How lean are the tool descriptions returned? Token waste ratio | 0.0–1.0 |
| `skill_gate_accuracy` | Does skill threshold filtering work correctly? | Binary (pass/fail) |
| `cold_start_latency` | Time to first result on a fresh Qdrant index | ms |
| `re_discovery_latency` | Time when index is warm and agent context is known | ms |

### Family B — Invocation Reliability

How reliably does `InvokeTool` execute handlers under various conditions?

| Test | Measures | Score |
|---|---|---|
| `success_rate` | Does the tool return expected output for known inputs? | Pass rate |
| `error_handling` | Structured errors on bad input (not crashes) | Pass rate |
| `streaming_latency` | Do streaming tools deliver all chunks without drops? | Chunk loss rate |
| `timeout_behavior` | Correct timeout → `ToolError` on expiry | Pass rate |
| `proof_quality` | For proof-handler tools: `final_score` quality dimension | 0.0–1.0 |
| `audit_completeness` | Every invocation logged with full metadata | 0.0–1.0 |
| `concurrency_safety` | Under N concurrent callers, results stay isolated | Pass rate |

### Family C — Skill & ACL Accuracy

How well do ACL enforcement and skill integration work?

| Test | Measures | Score |
|---|---|---|
| `acl_false_positive` | Are forbidden tools ever returned to unauthorized agents? | Rate (lower = better) |
| `acl_false_negative` | Are permitted tools ever incorrectly blocked? | Rate (lower = better) |
| `artifact_retrieval` | Do artifact_binding queries return semantically relevant artifacts? | 0.0–1.0 |
| `skill_gain_propagation` | Skill gain after task completion correctly updates the index | Latency + accuracy |
| `tier_unlock_correctness` | Tier unlocks triggered at right thresholds | Pass rate |

Both Casbin policy evaluation and SurrealDB `PERMISSIONS` RBAC are tested.

## DAP Server Score

Beyond per-tool scores, DAP Bench produces a **server-level DAP score** — a single number reflecting deployment quality:

```
dap_server_score = (
    discovery_precision_avg * 0.25
  + acl_accuracy            * 0.25   # hard requirement — 0.0 here = fail
  + invocation_reliability  * 0.20
  + audit_completeness      * 0.15
  + skill_integration_score * 0.15
)
```

ACL accuracy is weighted as a hard gate — a server that leaks forbidden tools to agents fails the benchmark regardless of other scores.

## Running DAP Bench

Install and run as a standard DAP Hub package:

```bash
# Install
dap-cli install core/dap-bench --target local

# Run full suite
dap-bench run --server grpc://localhost:50051 --suite full

# Run specific families
dap-bench run --families A,B --agent-id bench_agent_001

# Run against a specific tool
dap-bench run --tool port_scanner --families B,C

# Compare two servers (A/B test after config change)
dap-bench compare \
  --server-a grpc://localhost:50051 \
  --server-b grpc://localhost:50052 \
  --families A,B,C

# Output to JSON for CI integration
dap-bench run --server grpc://localhost:50051 --output results.json
```

## Leaderboard

DAP Bench scores are published on DAP Hub. Server implementations compete on efficiency — scores are comparable across deployments. Implementations with higher discovery precision, lower invocation latency, and stricter ACL enforcement rank higher.

## SurrealLife Integration

In SurrealLife, DAP Bench runs are **research company tasks**. A research company specializing in `domain: infrastructure` can be commissioned to benchmark a company's internal tool registry:

```
Commission: "Audit AcmeCorp's internal DAP tool registry"
  → Research company agents run dap-bench against acmecorp namespace
  → Produce benchmark report with per-tool and server-level scores
  → Embargoed delivery to AcmeCorp, or published publicly
  → Feeds directly into AcmeCorp's Company Infrastructure Score
```

DAP Bench score also affects **Agent Telecom tier pricing** — higher-scoring infrastructure companies negotiate better network rates.

---

> **References**
> - [dap_protocol.md SS24 — DAP Bench](../../planning/prd/dap_protocol.md)
> - [dap_protocol.md SS22 — Tool & Benchmark Evaluation](../../planning/prd/dap_protocol.md)

*See also: [apps.md](apps.md) | [dapnet.md](dapnet.md)*
