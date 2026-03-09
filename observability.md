# DAP Observability — Reference

DAP Observability combines three layers: **structured audit logs** (SurrealDB), **distributed traces** (Langfuse), and **guardrail validation** (Haystack). Together they give full visibility into every agent action — what ran, why it ran, whether the output was safe, and whether it was good enough.

> DAP Logs tell you *what happened*. Langfuse traces tell you *how the LLM got there*. Haystack guardrails tell you *whether the output is safe to use*.

---

## Architecture

```mermaid
graph TD
    subgraph Invocation["Tool Invocation"]
        IT["InvokeTool RPC"]
    end

    subgraph Guardrail["Haystack Guardrail Phase"]
        GI["Input Guardrail\nvalidate params before LLM"]
        GO["Output Guardrail\nvalidate result before return"]
    end

    subgraph LLM["LLM Phase (Langfuse-traced)"]
        LF["Langfuse Trace\nspan per LLM call + PoT"]
        POT["PoT Gate\nscores output quality"]
    end

    subgraph Storage["Storage"]
        SDB["SurrealDB\ntool_call_log (DAP Logs)"]
        LFB["Langfuse Backend\ntraces + spans + evals"]
    end

    IT --> GI
    GI -->|"pass"| LF
    GI -->|"block"| SDB
    LF --> POT
    POT --> GO
    GO -->|"pass"| SDB
    GO -->|"block"| SDB
    LF --> LFB
    POT --> LFB
```

Every invocation flows: **input guardrail → LLM (traced) → PoT gate → output guardrail → log**.

---

## Langfuse Integration

[Langfuse](https://langfuse.com) is an open-source LLM observability platform. DAP integrates it as a **trace exporter** — every `InvokeTool` call that runs an LLM phase becomes a Langfuse trace, with child spans per phase.

### Trace Structure

```
Trace: InvokeTool(market_analysis)
├── Span: input_guardrail         [0ms – 8ms]   ✓ pass
├── Span: artifact_fetch          [8ms – 31ms]  3 artifacts injected
├── Span: llm_phase               [31ms – 890ms]
│   ├── Generation: system_prompt  (312 tokens)
│   ├── Generation: user_prompt    (89 tokens)
│   └── Generation: completion     (201 tokens)  → PoT score: 78
├── Span: pot_gate                [890ms – 920ms] score 78 ≥ 65 ✓
├── Span: output_guardrail        [920ms – 931ms] ✓ pass
└── Score: pot_quality            78 / 100
```

### SDK Integration

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse(
    public_key=settings.LANGFUSE_PUBLIC_KEY,
    secret_key=settings.LANGFUSE_SECRET_KEY,
    host=settings.LANGFUSE_HOST,   # self-hosted or langfuse.com
)

class DAPWorkflowRunner:

    @observe(name="InvokeTool")
    async def invoke_tool(self, tool_name: str, params: dict, agent_id: str):
        # Attach metadata to trace
        langfuse_context.update_current_trace(
            name=f"InvokeTool:{tool_name}",
            user_id=agent_id,
            metadata={
                "tool_name": tool_name,
                "team_id": self.team_id,
                "params_hash": sha256(str(params)),  # never log raw params
            }
        )

        with langfuse_context.observe_span("input_guardrail"):
            result = await self.run_input_guardrail(tool_name, params)
            if result.blocked:
                langfuse_context.update_current_observation(level="WARNING", status_message=result.reason)
                raise GuardrailError(result.reason)

        artifacts = await self.fetch_artifacts(tool_name, agent_id)

        with langfuse_context.observe_span("llm_phase"):
            output = await self.run_llm_phase(tool_name, params, artifacts)

        pot_score = await self.run_pot_gate(tool_name, output)
        langfuse_context.score_current_trace("pot_quality", value=pot_score / 100)

        with langfuse_context.observe_span("output_guardrail"):
            validated = await self.run_output_guardrail(tool_name, output)

        return validated
```

### Trace–Log Correlation

Both DAP Logs (SurrealDB) and Langfuse share a **trace ID** — enabling a full join between the structured log (what + outcome) and the trace (how the LLM got there):

```python
trace_id = langfuse_context.get_current_trace_id()

# Write DAP Log with trace reference
await db.create("tool_call_log", {
    "agent_id": f"agent:{agent_id}",
    "team_id": f"team:{team_id}",
    "tool_name": tool_name,
    "op": "invoke",
    "params_hash": params_hash,
    "outcome": "success",
    "pot_score": pot_score,
    "latency_ms": elapsed_ms,
    "token_cost": token_count,
    "trace_id": trace_id,   # ← Langfuse trace reference
})
```

```surql
-- Join DAP logs with Langfuse trace IDs for post-hoc analysis
SELECT agent_id, tool_name, pot_score, trace_id, latency_ms
FROM tool_call_log
WHERE outcome = "pot_failed"
  AND created_at > time::now() - 7d
ORDER BY pot_score ASC;
-- Then fetch trace_id details from Langfuse API for deep inspection
```

---

## Langfuse Evaluation

Langfuse supports **dataset-based evaluation** — replaying past invocations against new model versions or prompts, then scoring with an LLM-as-judge.

### PoT as Inline Evaluation

PoT (Proof of Thought) already runs inline. Langfuse captures every PoT score as a **trace score** — making it queryable across time without any additional evaluator:

```python
# After PoT scoring — record in Langfuse
langfuse.score(
    trace_id=trace_id,
    name="pot_quality",
    value=pot_score / 100,    # 0.0 – 1.0
    comment=f"threshold={threshold}, retries={retry_count}"
)
```

### Dataset Evaluation

```python
# Build evaluation dataset from past DAP logs
dataset = langfuse.create_dataset("market_analysis_evals")

# Pull failed PoT invocations from SurrealDB
failed = await db.query("""
    SELECT * FROM tool_call_log
    WHERE tool_name = "market_analysis"
      AND outcome IN ["pot_failed", "success"]
      AND created_at > time::now() - 30d
    LIMIT 200
""")

for log in failed:
    langfuse.create_dataset_item(
        dataset_name="market_analysis_evals",
        input={"params_hash": log["params_hash"], "tool": log["tool_name"]},
        expected_output={"pot_score_min": 65},
        metadata={"original_pot_score": log["pot_score"]}
    )

# Run evaluation against new model version
for item in langfuse.get_dataset("market_analysis_evals").items:
    with item.observe(run_name="gpt-4o-vs-gemini-flash") as trace:
        output = await invoke_with_new_model(item.input)
        langfuse.score(trace_id=trace.id, name="pot_quality", value=output.pot_score / 100)
```

---

## Haystack Guardrails

[Haystack](https://haystack.deepset.ai) provides pipeline-based validation. In DAP, guardrails are a **workflow phase type** (`type: guardrail`) — executed before the LLM phase (input) and after PoT (output).

### Workflow Definition

```yaml
name: market_analysis
workflow: market_analysis_flow.yaml
```

```yaml
# market_analysis_flow.yaml
phases:
  - type: guardrail
    id: input_check
    direction: input
    pipeline: guardrails/market_input.yaml
    on_block: reject          # reject | warn | redact

  - type: rag
    id: artifact_fetch
    skill: finance
    top_k: 3

  - type: llm
    id: analysis
    model: gemini-2.0-flash
    prompt: prompts/market_analysis.jinja2

  - type: proof_of_thought
    id: pot
    threshold: 65
    max_retries: 2

  - type: guardrail
    id: output_check
    direction: output
    pipeline: guardrails/market_output.yaml
    on_block: reject
```

### Input Guardrail Pipeline

```yaml
# guardrails/market_input.yaml
components:
  - name: symbol_validator
    type: RegexValidator
    params:
      pattern: "^[A-Z]{2,10}$"
      field: symbol

  - name: injection_detector
    type: PromptInjectionDetector
    params:
      threshold: 0.85

  - name: pii_filter
    type: PIIDetector
    params:
      entities: [EMAIL, PHONE, SSN]
      action: block

pipeline:
  - symbol_validator
  - injection_detector
  - pii_filter
```

### Output Guardrail Pipeline

```yaml
# guardrails/market_output.yaml
components:
  - name: hallucination_check
    type: LLMEvaluator
    params:
      model: gemini-2.0-flash
      prompt: "Does this analysis cite only real, verifiable market data? Answer YES or NO."
      threshold: 0.8
      field: analysis_text

  - name: risk_disclosure_check
    type: KeywordPresenceChecker
    params:
      required: ["risk", "not financial advice"]
      action: warn    # warn only, don't block

  - name: length_guard
    type: LengthValidator
    params:
      min_tokens: 50
      max_tokens: 2000

pipeline:
  - hallucination_check
  - risk_disclosure_check
  - length_guard
```

### Python Integration

```python
from haystack import Pipeline
from haystack.components.validators import RegexValidator

class DAPGuardrailPhase:
    def __init__(self, pipeline_path: str, direction: str):
        self.pipeline = Pipeline.load(pipeline_path)
        self.direction = direction

    async def run(self, payload: dict) -> GuardrailResult:
        result = self.pipeline.run(payload)
        blocked = result.get("blocked", False)
        reason = result.get("reason", "")

        # Log guardrail decision
        await db.create("guardrail_log", {
            "direction": self.direction,
            "pipeline": self.pipeline_path,
            "blocked": blocked,
            "reason": reason,
            "tool_name": payload.get("tool_name"),
            "agent_id": payload.get("agent_id"),
            "created_at": datetime.utcnow(),
        })

        return GuardrailResult(blocked=blocked, reason=reason)
```

---

## Combined Observability Stack

```mermaid
graph LR
    subgraph Inbound["Inbound"]
        REQ["InvokeTool Request"]
    end

    subgraph Guard1["Input Guardrail (Haystack)"]
        IG["injection · PII · schema"]
    end

    subgraph Exec["Execution (Langfuse-traced)"]
        RAG["RAG Phase"]
        LLM["LLM Phase"]
        POT["PoT Gate"]
    end

    subgraph Guard2["Output Guardrail (Haystack)"]
        OG["hallucination · length · disclosure"]
    end

    subgraph Sink["Observability Sinks"]
        SDBL["SurrealDB\ntool_call_log + guardrail_log"]
        LFT["Langfuse\ntraces + scores + datasets"]
        MQ["MQTT\ndap/logs/stream"]
    end

    REQ --> IG
    IG -->|"pass"| RAG
    RAG --> LLM
    LLM --> POT
    POT --> OG
    OG -->|"pass / block"| SDBL
    OG --> MQ
    LLM --> LFT
    POT --> LFT
```

| Layer | Tool | What it captures |
|---|---|---|
| **Audit log** | SurrealDB `tool_call_log` | Outcome, params_hash, latency, token cost, PoT score |
| **Distributed trace** | Langfuse | LLM prompts, completions, token counts, span timing |
| **Evaluation** | Langfuse Datasets | PoT score trends, A/B model comparison, regression detection |
| **Input guardrail** | Haystack Pipeline | Injection, PII, schema violations — blocked before LLM |
| **Output guardrail** | Haystack Pipeline | Hallucination, length, required disclosures |
| **Stream** | MQTT | Real-time log feed for n8n, dashboards, alerting |
| **Alert rules** | SurrealDB DEFINE EVENT | Pattern-triggered escalation (repeated failures, farming) |

---

## SurrealDB Schema Extension

```surql
-- Guardrail log — separate table, linked to tool_call_log
DEFINE TABLE guardrail_log SCHEMAFULL PERMISSIONS
    FOR select WHERE $auth.team_id = team_id OR $auth.role CONTAINS "admin"
    FOR create NONE
    FOR update NONE
    FOR delete NONE;

DEFINE FIELD id          ON guardrail_log TYPE record;
DEFINE FIELD tool_name   ON guardrail_log TYPE string;
DEFINE FIELD agent_id    ON guardrail_log TYPE record<agent>;
DEFINE FIELD team_id     ON guardrail_log TYPE record<team>;
DEFINE FIELD direction   ON guardrail_log TYPE string;   -- input | output
DEFINE FIELD pipeline    ON guardrail_log TYPE string;
DEFINE FIELD blocked     ON guardrail_log TYPE bool;
DEFINE FIELD reason      ON guardrail_log TYPE option<string>;
DEFINE FIELD trace_id    ON guardrail_log TYPE option<string>;   -- Langfuse trace ref
DEFINE FIELD created_at  ON guardrail_log TYPE datetime DEFAULT time::now();

-- Add trace_id to existing tool_call_log
DEFINE FIELD trace_id ON tool_call_log TYPE option<string>;
```

---

## Alert: Repeated Guardrail Blocks

```surql
-- Alert on repeated input blocks for same agent (possible adversarial probing)
DEFINE EVENT guardrail_probe_detect ON guardrail_log
WHEN $event = "CREATE" AND $after.blocked = true AND $after.direction = "input" THEN {
    LET $recent_blocks = (
        SELECT count() FROM guardrail_log
        WHERE agent_id = $after.agent_id
          AND blocked = true
          AND direction = "input"
          AND created_at > time::now() - 1h
        GROUP ALL
    )[0].count;

    IF $recent_blocks >= 5 {
        http::post('http://dapnet/internal/alerts', {
            type:  "guardrail_probe_suspected",
            agent: $after.agent_id,
            count: $recent_blocks,
            last_reason: $after.reason,
        });
    };
};
```

---

## Deployment

### Self-Hosted Langfuse (Docker)

```yaml
# docker-compose.yml addition
langfuse:
  image: langfuse/langfuse:latest
  ports:
    - "3100:3000"
  environment:
    DATABASE_URL: postgresql://langfuse:secret@postgres:5432/langfuse
    NEXTAUTH_SECRET: your-secret
    SALT: your-salt
  depends_on:
    - postgres
```

```env
# .env — DAP server
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=http://langfuse:3100
LANGFUSE_ENABLED=true
```

### Disabling per Environment

```yaml
# Guardrail phase with disabled flag — skip in local dev
phases:
  - type: guardrail
    id: input_check
    enabled: ${GUARDRAILS_ENABLED:-true}
    pipeline: guardrails/market_input.yaml
```

---

> **References**
> - Langfuse (2024). *Open Source LLM Observability.* [langfuse.com](https://langfuse.com) — trace-level visibility into LLM calls; DAP uses Langfuse for per-span observability
> - deepset (2024). *Haystack 2.0 — Composable AI Pipelines.* [haystack.deepset.ai](https://haystack.deepset.ai) — modular pipeline components; DAP guardrail phase runs Haystack pipelines as validation gates
> - Breck et al. (2017). *The ML Test Score.* Google. — production ML validation principles; guardrail phases operationalize input/output validation at inference time

*See also: [logs.md](logs.md) · [proof-of-thought.md](proof-of-thought.md) · [workflows.md](workflows.md) · [surreal-events.md](surreal-events.md) · [n8n.md](n8n.md)*
*Full spec: [dap_protocol.md](../../planning/prd/dap_protocol.md)*
