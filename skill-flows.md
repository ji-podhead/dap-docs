# DAP Skill Flows — Reference

Skill Flows are the complete pipeline connecting skills, tools, RAG, workflows, and memory. Five independent flows cover the full skill lifecycle — from tool discovery through execution to knowledge gain.

> Skills gate what agents can see. Artifacts shape what they bring. Workflows define how they execute. PoT gates what gets delivered. Everything writes back into the skill store.

---

## The Full Pipeline (one InvokeTool call)

```
Agent activates
  │
  ├─ [Flow 1] DiscoverTools(context, agent_skills)
  │     → ACL filter → skill gate → semantic rank by bloat_score
  │     → Agent sees only tools they can use. High-skill tools invisible to low-skill agents.
  │
  ├─ Agent selects tool, calls InvokeTool("market_analysis", params)
  │
  ├─ [Flow 3] Pre-execution checks
  │     → ACL check (Casbin) → skill check → param validation
  │     → FAIL: ToolError returned, handler never runs
  │
  ├─ Artifact injection (before handler runs)
  │     → HNSW query: top-3 skill artifacts matching task context
  │     → injected into workflow as grounding (agent's accumulated approaches)
  │
  ├─ Workflow executes (market_analysis_flow.yaml):
  │     Phase 1 [rag]    → SurrealDB HNSW: 5 chunks, max 400 tokens, ACL auto
  │     Phase 2 [llm]    → task + grounding + skill artifacts → analysis
  │     Phase 3 [pot]    → score ≥ 65? pass : retry Phase 2 (max 2×)
  │     Phase 4 [script] → quantitative signals from agent's saved scripts
  │
  ├─ Result → artifact stored in SurrealDB + graph-linked
  │
  └─ [Flow 4] SkillGainEvent emitted in InvokeResponse
        → host applies gain to skill store
        → successful approach stored as new artifact
```

---

## Flow 1 — Activation: Skill Scores into DiscoverTools

```
Host: load agent skill scores → {"hacking": 42, "finance": 71}

DAP Server: DiscoverTools(agent_skills={...})
  → Casbin: filter by ACL roles
  → Skill gate: tool.skill_min vs agent_skills[tool.skill_required]
       attempt_hack_database (skill_min=60) → dropped (agent has 42)
       market_analysis       (skill_min=40) → kept   (agent has 71)
  → Qdrant: rank remaining by semantic similarity to context
  → Return: ToolSummary list (description_tokens only)

Agent LLM: sees only what it can use
  → "attempt_hack_database" does not exist in the agent's world
  → Not "permission denied" — the tool simply isn't in the list
```

**Why this matters:** no prompt leakage of unavailable tools. The agent's LLM cannot try to call a tool it doesn't know about. Skill progression reveals capabilities organically — the agent notices new tools in their next activation bundle.

---

## Flow 2 — Search: Skill-Filtered On-Demand Discovery

```
Agent calls SearchTools("I need to escalate privileges")
  → Embed query
  → Qdrant semantic search over tool_registry
  → Apply same ACL + skill filter as DiscoverTools
  → Return top-K matches

If no results: agent knows no matching tool exists for their current profile
  → Decision: train up the skill, or use a different approach
```

---

## Flow 3 — Invocation: Pre-Execution Checks

```
InvokeTool("attempt_hack_web", params, agent_skills={hacking: 42})
  │
  ├─ 1. ACL: casbin.enforce(agent_id, "/tools/attempt_hack_web", "call")
  │         FAIL → ToolError(permission_denied) — handler never runs
  │
  ├─ 2. Skill: agent_skills["hacking"] >= tool.skill_min (40)
  │         42 >= 40 → PASS
  │         FAIL → ToolError(skill_insufficient, hint="need hacking ≥ 40")
  │
  ├─ 3. Params: validate against tool schema
  │         FAIL → ToolError(invalid_params)
  │
  ├─ 4. Artifact injection:
  │         HNSW query on agent's skill_artifacts WHERE skill = "hacking"
  │         Top-3 by cosine similarity to task → injected into workflow context
  │
  ├─ 5. Dispatch handler (yaml / notebook / proof / crew / ...)
  │
  ├─ 6. Stream InvokeResponse chunks
  │
  └─ 7. Audit log: tool_call_log {agent_id, tool, params_hash, outcome, latency_ms}
```

---

## Flow 4 — Skill Gain: Post-Invocation Feedback Loop

```
DAP Server (after successful invocation):
  → Read tool registry: skill_linked = "hacking", skill_gain = 1.5
  → Emit SkillGainEvent in final InvokeResponse:
    {skill_name: "hacking", gain: 1.5, tool_name: "...", agent_id: "..."}

Host system (agent runtime):
  → Apply business rules (host owns the skill store, DAP only suggests gain):
      - only on outcome == "success"
      - cap daily gain to prevent farming
      - scale by PoT score if available: gain × (pot_score / 100)
  → Write updated skill score
  → Store successful workflow artifact in skill_artifact collection
  → Next DiscoverTools reflects new score automatically
```

**DAP does not mutate skill scores.** It emits the event. The host applies the write. DAP stays stateless with respect to skills — the host owns the truth.

---

## Flow 5 — Skill Tier Unlock: New Tools Appear

```
Host: agent hacking score crosses 40 (tier threshold)
  → Update skill store: hacking = 41

Next DiscoverTools(agent_skills={hacking: 41}):
  → "attempt_hack_web" (skill_min=40) now passes skill gate
  → Appears in DiscoverResponse for the first time

Agent LLM: sees a new capability in their context bundle
  → No tutorial, no flag, no notification
  → The world simply expanded
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

```
Attempt 1: analysis phase → PoT score 58 < 65 → retry
Attempt 2: analysis phase → PoT score 73 ≥ 65 → continue

If 2 retries exhausted and still < 65:
  → workflow fails with PoT_THRESHOLD_NOT_MET
  → partial result returned with pot_score: 52
  → host can decide: return to agent, escalate, or discard
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

```
Senior analyst (finance: 78) hired to produce market report:
  → DiscoverTools: sees 12 tools (junior sees 4)
  → Artifact injection: 3 proven strategies from their skill store injected
  → RAG phase: 400 tokens of current data (same as junior)
  → LLM phase: reasons with richer context than junior
  → PoT gate: passes first attempt (score: 81)
  → Result: proofed artifact, skill gain × 1.5
  → New approach stored as artifact → next time even better
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
