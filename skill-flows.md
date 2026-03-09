# DAP Skill Flows — Reference

Skill Flows are the complete pipeline connecting skills, tools, RAG, workflows, and memory. Five independent flows cover the full skill lifecycle — from tool discovery through execution to knowledge gain.

> Skills gate what agents can see. Artifacts shape what they bring. Workflows define how they execute. PoT gates what gets delivered. Everything writes back into the skill store.

---

## The Full Pipeline (one InvokeTool call)

```mermaid
graph TD
    A[Agent activates] --> B["Flow 1: DiscoverTools(context, agent_skills)"]
    B --> C[ACL filter]
    C --> D[Skill gate]
    D --> E[Semantic rank by bloat_score]
    E --> F[Agent selects tool]
    F --> G["InvokeTool('market_analysis', params)"]
    G --> H[Flow 3: Pre-execution checks]
    H --> H1{ACL check}
    H1 -->|FAIL| HE[ToolError returned]
    H1 -->|PASS| H2{Skill check}
    H2 -->|FAIL| HE
    H2 -->|PASS| H3{Param validation}
    H3 -->|FAIL| HE
    H3 -->|PASS| I[Artifact injection]
    I --> I1[HNSW query: top-3 skill artifacts]
    I1 --> J[Workflow executes]
    J --> J1["Phase 1 [rag]: SurrealDB HNSW, 5 chunks, 400 tokens"]
    J1 --> J2["Phase 2 [llm]: task + grounding + artifacts → analysis"]
    J2 --> J3{"Phase 3 [pot]: score >= 65?"}
    J3 -->|retry max 2x| J2
    J3 -->|PASS| J4["Phase 4 [script]: quantitative signals"]
    J4 --> K[Result stored as artifact in SurrealDB]
    K --> L[Flow 4: SkillGainEvent emitted]
    L --> M[Host applies gain to skill store]
    L --> N[Successful approach stored as new artifact]
```

---

## Flow 1 — Activation: Skill Scores into DiscoverTools

```mermaid
graph TD
    A[Host loads agent skill scores] --> B["agent_skills = {hacking: 42, finance: 71}"]
    B --> C["DAP Server: DiscoverTools(agent_skills)"]
    C --> D[Casbin: filter by ACL roles]
    D --> E{Skill gate: tool.skill_min vs agent score}
    E -->|"attempt_hack_database skill_min=60, agent has 42"| F[Dropped]
    E -->|"market_analysis skill_min=40, agent has 71"| G[Kept]
    G --> H[Qdrant: rank by semantic similarity to context]
    H --> I[Return ToolSummary list]
    I --> J[Agent LLM sees only tools it can use]
    J --> K["attempt_hack_database does not exist in agent's world"]
```

**Why this matters:** no prompt leakage of unavailable tools. The agent's LLM cannot try to call a tool it doesn't know about. Skill progression reveals capabilities organically — the agent notices new tools in their next activation bundle.

---

## Flow 2 — Search: Skill-Filtered On-Demand Discovery

```mermaid
graph TD
    A["Agent calls SearchTools('I need to escalate privileges')"] --> B[Embed query]
    B --> C[Qdrant semantic search over tool_registry]
    C --> D[Apply ACL + skill filter]
    D --> E{Results found?}
    E -->|Yes| F[Return top-K matches]
    E -->|No| G[Agent knows no matching tool exists for current profile]
    G --> H{Decision}
    H --> H1[Train up the skill]
    H --> H2[Use a different approach]
```

---

## Flow 3 — Invocation: Pre-Execution Checks

```mermaid
graph TD
    A["InvokeTool('attempt_hack_web', params, agent_skills={hacking:42})"] --> B
    B["1. ACL: casbin.enforce(agent_id, path, 'call')"] -->|FAIL| E1["ToolError: permission_denied"]
    B -->|PASS| C
    C["2. Skill: agent_skills['hacking'] >= tool.skill_min (40)"] -->|FAIL| E2["ToolError: skill_insufficient"]
    C -->|"42 >= 40 PASS"| D
    D[3. Params: validate against tool schema] -->|FAIL| E3["ToolError: invalid_params"]
    D -->|PASS| F
    F["4. Artifact injection: HNSW top-3 by cosine similarity → injected into workflow"] --> G
    G["5. Dispatch handler (yaml / notebook / proof / crew)"] --> H
    H[6. Stream InvokeResponse chunks] --> I
    I["7. Audit log: tool_call_log {agent_id, tool, params_hash, outcome, latency_ms}"]
```

---

## Flow 4 — Skill Gain: Post-Invocation Feedback Loop

```mermaid
graph TD
    A[DAP Server: successful invocation] --> B["Read tool registry: skill_linked='hacking', skill_gain=1.5"]
    B --> C["Emit SkillGainEvent in InvokeResponse: {skill_name, gain, tool_name, agent_id}"]
    C --> D[Host system receives event]
    D --> E{outcome == success?}
    E -->|No| Z[Discard event]
    E -->|Yes| F[Apply business rules]
    F --> F1[Cap daily gain to prevent farming]
    F --> F2["Scale by PoT score: gain x (pot_score / 100)"]
    F1 --> G[Write updated skill score]
    F2 --> G
    G --> H[Store workflow artifact in skill_artifact collection]
    H --> I[Next DiscoverTools reflects new score automatically]
```

**DAP does not mutate skill scores.** It emits the event. The host applies the write. DAP stays stateless with respect to skills — the host owns the truth.

---

## Flow 5 — Skill Tier Unlock: New Tools Appear

```mermaid
graph TD
    A[Agent hacking score crosses threshold 40] --> B[Host updates skill store: hacking = 41]
    B --> C["Next DiscoverTools(agent_skills={hacking: 41})"]
    C --> D{"attempt_hack_web (skill_min=40): 41 >= 40?"}
    D -->|PASS| E[Tool appears in DiscoverResponse for the first time]
    E --> F[Agent LLM sees new capability in context bundle]
    F --> G[No tutorial, no flag — the world simply expanded]
```

---

## RAG Phase in Skill Flows

The `type: rag` phase is how workflows ground themselves in current knowledge — distinct from artifact injection (which is past experience):

```yaml
# Inside any skill workflow YAML
- id: ground_context
  type: rag
  source: surreal
  collections:
    - web_content_public              # current market data, news
    - "agent_memory_{{ agent_id }}"   # agent's own past findings
    - "skill_artifacts_{{ skill }}"   # domain knowledge from skill store
  query_from: task.input
  top_k: 5
  max_tokens: 400          # hard budget
  summarize: true
  persist_links: true      # RELATE agent->fetched->web_content
  access_filter: auto      # SurrealDB PERMISSIONS fire automatically
```

**Artifact injection (Flow 3) vs RAG phase:**

| | Artifact Injection | RAG Phase |
|---|---|---|
| Source | Agent's skill_artifact collection | Any SurrealDB HNSW collection |
| Timing | Before workflow starts | During workflow (explicit phase) |
| Content | Past proven approaches, scripts, templates | Current grounding: news, web, memories |
| Token budget | Implicit (top_k artifacts) | Explicit `max_tokens` hard limit |
| Persistence | Already stored | `persist_links: true` → graph-linked after retrieval |

An experienced agent gets both: past approaches injected before the workflow, plus current grounding during the RAG phase. Their context is richer at both ends.

---

## PoT Gate in Skill Flows

After an `llm` phase, a `proof_of_thought` gate checks output quality before proceeding:

```yaml
- id: verify_analysis
  type: proof_of_thought
  input_from: [analysis]
  score_threshold: 65       # 0–100
  retry_phase: analysis     # re-run if below threshold
  max_retries: 2
  emit_score: true          # PoT score attached to result artifact
```

```mermaid
graph TD
    A[analysis phase] --> B{PoT score >= 65?}
    B -->|"Attempt 1: score 58 < 65"| C[retry]
    C --> A
    B -->|"Attempt 2: score 73 >= 65"| D[continue to next phase]
    B -->|"2 retries exhausted, still < 65"| E[workflow fails: PoT_THRESHOLD_NOT_MET]
    E --> F["partial result returned with pot_score: 52"]
    F --> G{Host decides}
    G --> G1[Return to agent]
    G --> G2[Escalate]
    G --> G3[Discard]
```

A workflow that passes PoT produces a `proofed: true` artifact — 1.5× skill gain multiplier, higher rank in future HNSW queries, audit-grade in contracts.

---

## Skill Flows in SurrealLife

In SurrealLife, skill flows become the economic unit of work:

- A company hires agents based on `public.skill.score` (they can't see private artifacts)
- The agent's private artifacts shape how they actually execute — invisible competitive advantage
- A PoT-verified delivery earns premium contract rates — the proof is on-chain
- Agents with high skills attract better subagent talent (employment graph IS the permission)
- Skill depreciation (unused skills decay) creates continuous demand for university courses

```mermaid
graph TD
    A["Senior analyst (finance: 78) hired for market report"] --> B["DiscoverTools: sees 12 tools (junior sees 4)"]
    B --> C[Artifact injection: 3 proven strategies from skill store]
    C --> D["RAG phase: 400 tokens of current data"]
    D --> E["LLM phase: reasons with richer context than junior"]
    E --> F{"PoT gate: score >= 65?"}
    F -->|"First attempt: score 81"| G["Result: proofed artifact, skill gain x 1.5"]
    G --> H[New approach stored as artifact]
    H --> I[Next time: even better context]
```

---

## Error Cases

| Error | When | Agent sees |
|---|---|---|
| `skill_insufficient` on Invoke | Agent directly calls a tool with too-low skill | Structured error with skill gap: "need hacking ≥ 60, have 42" |
| Tool absent from SearchTools | Skill below visibility threshold | No results — tool doesn't exist to the agent |
| PoT threshold not met | Output quality below `score_threshold` after `max_retries` | `PoT_THRESHOLD_NOT_MET` + partial result with score |
| Skill score stale | Host skill store lagging | Old score used — tool may be blocked despite real qualification |
| Skill provider down | `http:{url}` provider unreachable | Falls back to `skill_gating_fallback`: `allow_all` / `deny_skill_gated` / `error` |
| Subagent not employed | `type: subagent` phase with unappointed agent | `SUBAGENT_NOT_IN_EMPLOYMENT_GRAPH` — hire them first |

---

> **References**
> - Yao et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models.* ICLR 2023. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629) — reasoning + action interleaved; skill flows operationalize this as typed workflow phases
> - Shinn et al. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning.* NeurIPS 2023. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366) — self-improvement via verbal feedback; PoT retry loop is the structured equivalent
> - Wang et al. (2024). *A Survey on Large Language Model based Autonomous Agents.* [arXiv:2308.11432](https://arxiv.org/abs/2308.11432) — skill memory and tool-use in agent architectures

*See also: [skills.md](skills.md) · [workflows.md](workflows.md) · [rag.md](rag.md) · [artifacts.md](artifacts.md) · [proof-of-thought.md](proof-of-thought.md)*
*Full spec: [dap_protocol.md §12](../../planning/prd/dap_protocol.md)*
