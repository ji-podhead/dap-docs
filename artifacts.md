# DAP Artifacts — Reference

Artifacts are the **executable knowledge units** of the DAP skill system. Every tool invocation, workflow completion, and mentorship grant produces artifacts — stored in SurrealDB, embedded in Qdrant, and linked via graph edges. They are what make a skilled agent different from a fresh one.

## What Is an Artifact?

An artifact is any reusable output of agent work: a script, a workflow template, a query, a crew config. Artifacts live in the agent's skill store and are retrieved by semantic similarity when a related tool is invoked. An agent with 50 completed tasks has 50 artifacts to draw from — their context is richer, their execution is better.

## Artifact Structure

```python
{
    "id":            "skill_artifact:ulid",
    "tool_name":     "pentest_webapp",
    "agent_id":      "agent:alice",
    "skill":         "hacking",
    "type":          "workflow",           # script | workflow | query | crew_yaml | regex
    "content":       "<yaml or python>",
    "context_description": "Multi-phase API security audit for REST endpoints",
    "tags":          ["api", "security", "rest"],
    "quality_score": 0.82,
    "pot_score":     78,                   # PoT score if proof_of_thought phase ran
    "proofed":       True,                 # PoT passed threshold
    "source":        "task_completion",    # task_completion | mentorship | university | self_authored
    "embedding":     [0.012, -0.034, ...], # HNSW vector for semantic retrieval
    "created_at":    "2025-09-14T10:24:03Z"
}
```

## Binding Modes

When a tool declares `artifact_binding` in its registration YAML, DAP fetches matching artifacts at invocation time. Three binding modes control how artifacts reach the handler:

| Mode | How it works | When to use |
|---|---|---|
| `inject` (default) | Artifacts injected into handler context at `inject_as` path | Notebook/YAML handlers that read artifacts directly |
| `prepend_prompt` | Artifacts prepended to LLM prompt as examples | LLM-based tools that need few-shot context |
| `select_workflow` | Highest-ranked artifact IS the execution template | Tool acts as dispatcher -- artifact defines the steps |

```yaml
artifact_binding:
  - skill: hacking
    artifact_types: [script, workflow, query]
    match_query: "webapp pentest reconnaissance"
    top_k: 3
    inject_as: "agent_context.hacking_artifacts"
```

## `select_workflow` Mode

The most powerful binding mode. The tool itself becomes a workflow runner -- the tool registry entry says "run whichever workflow template from this skill best matches the invocation context." The agent's accumulated templates compete semantically:

- **Junior agent** (few templates) -- gets a generic fallback or no match.
- **Senior agent** (rich template library) -- gets their best proven approach automatically selected.

The highest-ranked artifact IS the execution template for the next run. This is how skill scores translate into real capability differences without hardcoding tier-specific behavior.

## Artifact Accumulation

Every successful task can submit its approach as a new artifact:

```
POST /admin/agents/{agent_id}/skills/{skill_name}/artifacts
Body: {
  type: "workflow",
  content: "<yaml or python content>",
  context_description: "Multi-phase API security audit for REST endpoints",
  tags: ["api", "security", "rest"],
  source: "task_completion",
  quality_score: 0.82
}
```

The agent runtime calls this endpoint as part of skill gain recording. The artifact is embedded in the agent's skill Qdrant collection, ranked by quality score. Running a tool 50 times builds a library of 50 proven approaches -- each one retrievable by semantic similarity for future invocations.

**Skill gain and artifact accumulation are simultaneous.** When an agent earns score points, the skill store receives the successful approach as a new artifact. Both the number and the knowledge grow together. Skill decay (from neglect) means artifact relevance scores also decay -- stale approaches are down-weighted in injection ranking.

## Artifact Injection at Workflow Start

When DAP invokes a skill-linked tool:

```
InvokeTool("pentest_webapp", params={target: "alphastack.agentnet"}, agent_skills={"hacking": 65})
  |
  v
DAP server:
  1. Score check: 65 >= skill_min(40) --> pass
  2. Artifact fetch: HNSW query top-3 hacking artifacts matching "webapp pentest"
  3. Inject artifacts into handler context alongside params
  4. Handler executes with params + agent's accumulated approach library
  |
  v
Result reflects the agent's accumulated experience, not just a generic tool response
```

An agent with `hacking: 20` gets the tool but no injected artifacts -- they execute from scratch. An agent with `hacking: 80` gets the tool and a rich library of tested approaches. **The skill gap is not just access -- it is execution quality.**

## Graph Linking

Artifacts are connected to the rest of the system via SurrealDB graph edges:

```surql
-- Agent created this artifact
RELATE agent:alice->created->skill_artifact:ulid SET
    created_at = time::now(),
    context = "task completion";

-- Artifact was used in a task
RELATE skill_artifact:ulid->used_in->task:sprint_42 SET
    injected_at = time::now(),
    binding_mode = "select_workflow";
```

Graph traversal reconstructs the full provenance: which agent created the artifact, which tasks used it, what outcomes resulted.

## Artifact Inheritance

Artifacts are not always private. Five inheritance tiers control visibility:

| Source | Scope | Revoked on? | Who can see? |
|---|---|---|---|
| Agent's own artifacts | private | Never | Agent + employer |
| Company SOPs | company-public | Employment ends | All employees |
| Mentor grant | private-shared | Mentor revokes | Grantee only |
| University cert | public | Never (certified) | Anyone |
| Parent company | company-public | Acquisition reversed | Subsidiary employees |

Company SOPs are shared artifacts -- when an agent is employed, company artifacts appear alongside their own via the employment graph. When employment ends, access is revoked. IP theft detection: if artifacts appear in a competitor's crew after an agent leaves, the `->granted_by->` relation is evidence.

```surql
-- Mentor grants junior access to specific private artifacts
CREATE skill_grant SET
    from_agent  = agent:senior_alice,
    to_agent    = agent:junior_bob,
    skill       = "hacking",
    artifact_ids = ["port_scan_v2.py", "recon_flow.yaml"],
    expires_at  = sim::now() + sim::months(3),
    revocable   = true;
```

## `proofed: true` Effects

When a Proof of Thought phase scores an artifact above its threshold:

| Effect | Value |
|---|---|
| Skill gain multiplier | 1.5x |
| Artifact rank in skill store | Higher (used first in future crews) |
| Hub badge | `[PoT Verified]` shown on skill |
| Contract grade | Audit-grade -- legally binding in SurrealLife |
| `select_workflow` priority | Preferred over non-proofed templates |

Proofed artifacts are legally binding in-sim. If a research company delivers a `proofed: true` report under contract, disputes are resolved by the graph evidence -- not by agent claims.

## Workflow Artifacts with Phase Markers

Workflow artifacts are YAML templates that can include SimEngine phase markers:

```yaml
name: full_pentest_engagement
phases:
  - id: recon
    type: llm
    prompt_template: "Analyze {target} and identify attack surface..."

  - id: scan
    type: script
    script: "port_scan.py"
    args: {target: "{target}", timeout: 30}

  - id: sim_wait
    type: simengine
    duration_sim_hours: 2
    event: "target_scanned"

  - id: exploit
    type: llm
    input_from: scan

  - id: report
    type: crew
    crew_yaml: "pentest_report_crew.yaml"
    inputs: [recon, scan, exploit]
```

DAP executes phase by phase. SimEngine phases suspend the tool and resume after sim-time elapses. LLM phases invoke the agent's model; script phases run in sandbox; crew phases spawn sub-crews.

---
> **References**
> - Park et al. (2023). *Generative Agents: Interactive Simulacra of Human Behavior.* UIST 2023. [arXiv:2304.03442](https://arxiv.org/abs/2304.03442) -- memory retrieval and experience accumulation in agent systems
> - Shinn et al. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning.* NeurIPS 2023. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366) -- agents learning from past task outcomes
> - Packer et al. (2023). *MemGPT: Towards LLMs as Operating Systems.* [arXiv:2310.08560](https://arxiv.org/abs/2310.08560) -- hierarchical memory management for LLM agents

*Full spec: [dap_protocol.md SS10, SS12](../../planning/prd/dap_protocol.md)*
