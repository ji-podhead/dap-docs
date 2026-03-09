# DAP Crew Memory — Reference

In DAP, every CrewAI crew member can be backed by a real SurrealDB agent record — loading their accumulated memories and skill artifacts at initialization. In SurrealLife, this is mandatory (agents are persistent identities). In standalone deployments, it is optional but recommended for persistent agent teams.

## The Difference from Generic CrewAI

```python
# Generic CrewAI — static, no history
Agent(role="Analyst", backstory="You are a financial analyst.")

# DAP SurrealLife — dynamic, memory-backed
Agent(role=agent["role"], backstory=build_backstory(agent, memories, artifacts))
# backstory includes real past experiences + proven approaches
```

## Initialization Flow

```python
async def run_crew_phase(phase_config, task, db):
    crew_members = []
    task_vec = embed(task)

    for member_id in phase_config["members"]:
        # 1. Load agent record
        agent = await db.select(f"agent:{member_id}")

        # 2. Relevant memories via HNSW
        memories = await db.query("""
            SELECT context, outcome, pnl,
                   vector::similarity::cosine(embedding, $task_vec) AS score
            FROM agent_memory
            WHERE agent_id = $agent_id
              AND access_level IN $auth.access_levels
            ORDER BY score DESC LIMIT 5
        """, vars={"agent_id": member_id, "task_vec": task_vec})

        # 3. Top skill artifacts
        artifacts = await db.query("""
            SELECT content, context_description, quality_score
            FROM skill_artifact
            WHERE agent_id = $agent_id
            ORDER BY vector::similarity::cosine(embedding, $task_vec) DESC LIMIT 3
        """, vars={"agent_id": member_id, "task_vec": task_vec})

        # 4. Build dynamic backstory (Jinja template)
        backstory = render_jinja("backstory.md.j2", {
            "agent": agent, "memories": memories, "artifacts": artifacts,
            "inherited_artifacts": get_company_sops(agent, task_vec, db)  # get_company_sops() = [SurrealLife only] — returns empty list in non-SurrealLife deployments
        })

        # 5. CrewAI Agent with SurrealDB memory backend
        crew_members.append(CrewAI_Agent(
            role=agent["role"], goal=agent["goal"],
            backstory=backstory,
            memory=True,
            memory_config=SurrealMemoryBackend(agent_id=member_id, db=db)
        ))

    crew = Crew(agents=crew_members, tasks=build_tasks(phase_config, task))
    result = await crew.kickoff()

    # 6. Write memories back to all members
    for member_id in phase_config["members"]:
        await db.create("agent_memory", {
            "agent_id": member_id,
            "context": task,
            "outcome": result.summary,
            "quality_score": result.quality,
            "embedding": embed(f"{task} {result.summary}"),
            "session_id": current_session_id
        })

    return result
```

## SurrealMemoryBackend

Implements CrewAI's memory interface using SurrealDB HNSW. CrewAI's in-task memory reads/writes go directly to the agent's SurrealDB collection.

```python
class SurrealMemoryBackend:
    def __init__(self, agent_id: str, db: Surreal):
        self.agent_id = agent_id
        self.db = db

    async def save(self, text: str, metadata: dict):
        vec = embed(text)
        await self.db.create("agent_memory", {
            "agent_id": self.agent_id,
            "content": text,
            "embedding": vec,
            **metadata
        })

    async def search(self, query: str, limit: int = 5) -> list:
        vec = embed(query)
        return await self.db.query("""
            SELECT content, metadata,
                   vector::similarity::cosine(embedding, $vec) AS score
            FROM agent_memory
            WHERE agent_id = $agent_id
            ORDER BY score DESC LIMIT $limit
        """, vars={"agent_id": self.agent_id, "vec": vec, "limit": limit})
```

No ChromaDB, no Redis, no separate vector store — SurrealDB handles everything.

## Memory Access Control

Each crew member only sees their own memories:

```surql
DEFINE TABLE agent_memory PERMISSIONS
  FOR select WHERE agent_id = $auth.id
  FOR create WHERE agent_id = $auth.id;
```

A junior analyst in the same crew as a senior analyst cannot read the senior's private memories — even when they share a session.

## The Virtuous Cycle

```
Agent assigned to crew
  → loads 5 most relevant past experiences
  → loads 3 best skill artifacts for this task type
  → executes with richer context than a fresh agent
  → outcome written back as new memory
  → quality score updates skill score
  → successful approach stored as new artifact
  → next time: even richer context
```

An agent with 50 crew experiences executes measurably better than one with 0. Not because their LLM is different — because their **context is richer**.

## Company SOPs in Crews [SurrealLife only]

Company SOPs require the SurrealLife employment graph (`->works_for->` relation). In a standard DAP deployment without company structures, this section does not apply — agents only have their own private artifacts.

Inherited company artifacts appear in the backstory alongside private artifacts. When an agent leaves a company, the SOPs vanish from their next crew context automatically (employment graph relation removed).

A company's workflow templates are a competitive advantage that compounds over time as employees' memories grow around them.

---
> **References**
> - Park et al. (2023). *Generative Agents: Interactive Simulacra of Human Behavior.* UIST 2023. [arXiv:2304.03442](https://arxiv.org/abs/2304.03442) — memory-reflection-planning loop for persistent agents
> - Zhong et al. (2024). *MemoryBank: Enhancing Large Language Models with Long-Term Memory.* AAAI 2024. [arXiv:2305.10250](https://arxiv.org/abs/2305.10250) — vector memory retrieval for long-horizon agent tasks
> - Hong et al. (2023). *MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework.* [arXiv:2308.00352](https://arxiv.org/abs/2308.00352) — role-based crew execution with shared memory

*Full spec: [dap_protocol.md §12](../../planning/prd/dap_protocol.md)*
