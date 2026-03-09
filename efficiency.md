# DAP Token Efficiency — Reference

DAP treats token usage as a **first-class protocol metric**, not an afterthought. Every tool, artifact, and workflow phase has a measured cost. The system gates on quality and optimizes for signal density.

> MCP problem: 50 tools × ~200 tokens/schema = 10,000 tokens before the agent has done anything.
> DAP answer: discover 4 tools relevant to this task × ~10 tokens/summary = ~40 tokens.

---

## The Numbers

### MCP baseline (typical production setup)
```
Session start:
  50 tool schemas injected into system prompt   →  8,000 tokens
  RAG: 5 raw chunks × 300 tokens each           →  1,500 tokens
  ─────────────────────────────────────────────────────────────
  Total before agent does anything              → ~10,000 tokens
  Skill-adjusted context                        →      0 tokens (not supported)
  Quality gate on output                        →     none
```

### DAP (same task)
```
Session start:
  DiscoverTools("analyze BTC market conditions")
    → 4 tools match, summary_tokens only        →    ~40 tokens

Task execution (market_analysis workflow):
  Phase [rag]:   5 chunks summarized → injected →   ~200 tokens
  Phase [llm]:   task + grounding + 3 artifacts →   ~600 tokens total context
  Phase [pot]:   quality gate — retry if < 65   →     0 extra tokens if pass
  Phase [script]: runs analyst's saved script   →     0 LLM tokens
  ─────────────────────────────────────────────────────────────
  Total                                         →   ~900 tokens
  Skill-adjusted context                        →   yes — expert agent gets richer artifacts
  Quality gate on output                        →   PoT threshold enforced
```

**10,000 → 900 tokens. Same task. Better output for experienced agents.**

---

## bloat_score — Per-Tool Token Budget

Every tool in the DAP registry has a `bloat_score` — the estimated token cost of loading it at each stage:

```python
bloat_score = {
    "description_tokens":  18,   # name + one-line summary (DiscoverTools response)
    "schema_tokens":       94,   # full param schema (GetToolSchema, only if called)
    "artifact_tokens":    210,   # typical skill artifact injected for this tool type
    "total":              322    # worst-case full load
}
```

`DiscoverTools` injects only `description_tokens` per result. The agent calls `GetToolSchema` only for the tool they intend to invoke. Skill artifacts are injected only when execution starts.

**Ranking formula:**
```
discovery_rank = relevance_score × (1 − bloat_weight × (description_tokens / budget))
```

A tool with identical relevance but higher bloat_score ranks lower. Token efficiency is a competitive advantage in discovery.

### Bloat Score Validation

Tools are scored at registration time:

```yaml
# Tool YAML — bloat_score is auto-computed at registration
name: market_analysis
description: "Analyze market conditions for a symbol"   # 7 words — good
parameters:
  symbol: {type: string}
  timeframe: {type: string, enum: ["5m","1h","4h","1d"]}

# Auto-computed:
bloat_score:
  description_tokens: 12
  schema_tokens:      38    # enum values add tokens — accepted
  artifact_tokens:   180
  total:             230
  grade: A           # A=lean, B=acceptable, C=verbose, D=rejected
```

Tools graded `D` are rejected at registration — cannot enter the registry. Tools with grade `C` get a warning and cannot be featured tools on the DAP Hub.

---

## Validation Stack

DAP validates token efficiency at three levels:

### 1. Tool Registration Validation
```
POST /dap/tools/register
  → bloat_score computed
  → grade D → rejected (422)
  → grade C → warning, stored with flag
  → grade A/B → accepted
  → DiscoverTools ranking updated
```

### 2. PoT Quality Gate (per-invocation)
Every `llm` phase in a workflow can be followed by a `proof_of_thought` gate:

```yaml
- id: verify_quality
  type: proof_of_thought
  input_from: [analysis]
  score_threshold: 65
  retry_phase: analysis
  max_retries: 2
```

If the output scores below 65, the LLM phase reruns — not the entire workflow. A failed PoT gate costs tokens, but prevents a low-quality result from being delivered. **Output quality is enforced, not hoped for.**

PoT retry cost model:
```
Attempt 1:  600 tokens  → score 58 → retry
Attempt 2:  600 tokens  → score 71 → pass
PoT eval:   ~50 tokens  × 2 evals = 100 tokens
Total:      ~1,300 tokens for a verified result
vs. MCP:    ~10,000 tokens for an unverified result
```

### 3. PoS Search Efficiency Scoring
Proof of Search scores every search session against an optimal path:

```python
search_efficiency = min(100, (optimal_searches / actual_searches) * 100)
token_efficiency  = min(100, (optimal_tokens  / actual_tokens)  * 100)
path_efficiency   = (useful_searches / total_searches) * 100  # dead ends penalized

final_score = pot_score * 0.50 + evidence_quality * 0.20 + efficiency_score * 0.30
```

An agent that reaches the correct conclusion in 2 searches scores 100% search_efficiency. An agent that wastes 8 searches on dead ends scores low — and this feeds back into their `research` skill score. **The protocol incentivizes efficient reasoning.**

---

## Skill × Efficiency Compounding

The efficiency gains compound with agent experience:

```
Agent: financial_analysis skill = 10 (new)
  Discovery:  4 tools → 40 tokens
  RAG:        5 web chunks summarized → 200 tokens
  Artifacts:  0 (no skill store yet)
  LLM input:  task + 200 tokens grounding
  Output:     generic analysis
  Total:      ~500 tokens, C-grade output

Agent: financial_analysis skill = 75 (experienced)
  Discovery:  4 tools → 40 tokens
  RAG:        5 web chunks summarized → 200 tokens
  Artifacts:  3 proven strategies injected → 180 tokens
  LLM input:  task + 200 grounding + 180 expert artifacts
  Output:     expert analysis leveraging past approaches
  Total:      ~800 tokens, consistently A-grade output
```

More tokens spent on the experienced agent — but the quality delta is not marginal. The artifacts encode proven strategies that a fresh agent would spend 10x the tokens to rediscover via trial and error.

---

## DAP Bench — Measuring Efficiency

DAP Bench Family A measures token efficiency directly:

| Metric | What it measures |
|---|---|
| `discovery_token_cost` | Avg tokens consumed by DiscoverTools per task type |
| `schema_fetch_rate` | What % of discovered tools get GetToolSchema called — lower is better |
| `rag_chunk_utilization` | Ratio of injected RAG tokens that appear in the output reasoning |
| `pot_pass_rate` | % of llm phases that pass PoT gate on first attempt |
| `retry_token_overhead` | Avg extra tokens from PoT retries |
| `artifact_hit_rate` | % of invocations where a skill artifact was injected |

A DAP server gets a **DAP Efficiency Score** — published on the DAP Hub, comparable across implementations.

---

## DAP vs MCP vs Claude Code

| | Claude Code / MCP | DAP |
|---|---|---|
| **Tool loading** | All schemas in prompt at start | Semantic discovery at task time |
| **Tool budget** | No limit — grows with tool count | `bloat_score` enforced, grade D rejected |
| **RAG** | Chunk dump, no budget | `max_tokens` hard limit, summarized |
| **RAG access control** | Custom middleware | SurrealDB PERMISSIONS automatic |
| **Output quality gate** | None | PoT threshold — retry or fail |
| **Anti-hallucination** | None | PoS — Z3-verified evidence chain |
| **Agent experience** | Same context every session | Skill artifacts accumulated, HNSW-retrieved |
| **Persistence** | Session ends → gone | Graph-linked, retrievable forever |
| **Token cost (same task)** | ~10,000 | ~900 |
| **Quality validation** | User-observed | Protocol-enforced (PoT score, grade) |
| **Efficiency metric** | None | `bloat_score`, DAP Bench, PoS scorer |

---

## In SurrealLife — Efficiency as Economy

Token efficiency isn't just a technical metric — it's an economic one. Agent Telecom charges per-message fees on DAPNet. An agent that burns 10x the tokens on the same task pays 10x more in network costs. This creates:

- **Market pressure** toward lean tools (high bloat_score tools price themselves out)
- **Skill as asset** — experienced agents are cheaper to operate (artifact-backed reasoning)
- **PoS as premium tier** — verified research costs more compute, but commands higher contract prices
- **Tool competition** — two tools doing the same job, the leaner one wins on the market

DAP Bench scores are public. Agents and companies can compare tool implementations. A tool marketplace emerges from efficiency pressure.

---

> **References**
> - Shinn et al. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning.* NeurIPS 2023. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366) — self-improvement loop analogous to artifact accumulation from task outcomes
> - Yao et al. (2023). *Tree of Thoughts: Deliberate Problem Solving with Large Language Models.* NeurIPS 2023. [arXiv:2305.10601](https://arxiv.org/abs/2305.10601) — structured reasoning paths; PoT gate selects high-quality reasoning branches
> - Zhou et al. (2023). *Efficient Prompting via Dynamic In-Context Learning.* [arXiv:2305.11170](https://arxiv.org/abs/2305.11170) — dynamic context selection; bloat_score operationalizes this at protocol level
> - AgentSociety (2025). *Large-Scale Agent Simulation.* [arXiv:2502.08691](https://arxiv.org/abs/2502.08691) — MQTT at 10,000+ agent scale; efficiency pressure in multi-agent systems

*Full spec: [dap_protocol.md §3, §12b](../../planning/prd/dap_protocol.md)*
