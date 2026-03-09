# DAP Skills — Reference

> **Protocol vs Game:** Skill gates, gain events, and artifact memory are **DAP protocol features** — they work in any deployment. Boss endorsements, mentor grants, company inheritance, and career levels are **SurrealLife game-layer features**. See [dap-games.md](dap-games.md) for the full split.

Skills in DAP are not just scores. They are a structured knowledge store with public visibility, private artifacts, inheritance mechanics, and a derived score that no one can directly manipulate.

## Structure

```
skill
├── public/               ← visible to employers, ACL gates, endorsers
│   ├── score: 0–100      ← derived, never directly written
│   ├── level             ← novice / junior / mid / senior / expert
│   ├── certifications[]  ← sim-verifiable
│   ├── endorsed_by[]     ← PM/boss endorsements with weight
│   └── description
└── private/              ← agent + current employer only
    ├── artifacts[]       ← scripts, workflows, queries, crew YAMLs
    ├── memories[]        ← refs to agent_memory records
    ├── performance_log[] ← per-task quality scores (employer-appended)
    └── strategies[]      ← agent-authored notes
```

## Score Derivation

Score is **never directly written** — always computed:

```
score = base_score * 0.7
      + avg(endorsement.weight * pm_skill_weight) * 0.3

base_score updates after each task:
  new_score = old_score + (quality_score - 0.5) * learning_rate
  quality_score from PoT scorer (0–1)
  positive task → up, negative → down, slow decay over time
```

## Adaptive Learning

When conditions change, agents adapt through three mechanisms — no direct score override needed:

### 1. Adaptive Learning Rate

`learning_rate` is configurable per agent and per dimension. Higher rate = faster adaptation:

```surql
UPDATE agent SET
    skill_config.finance.learning_rate = 0.25,  -- default: 0.1, higher = faster adapt
    skill_config.finance.decay_rate    = 0.02   -- score decay per idle day
WHERE id = $agent_id;
```

An operator can temporarily raise `learning_rate` when deploying an agent into a new domain — old knowledge decays faster, new experience has more weight.

### 2. Regime Shift Signal

An agent can emit a `SkillRegimeShift` event when it detects its artifacts are no longer working (e.g. PoT scores consistently below threshold):

```python
# Agent-side: detect regime shift
if rolling_avg_pot_score < 0.4 and window_size >= 10:
    await dap.emit(SkillRegimeShift(
        agent_id=self.id,
        dimension="finance",
        reason="pot_scores_degraded",
        suggested_action="raise_learning_rate"
    ))
```

The DAP server handles this by:
- Temporarily raising `learning_rate` for that dimension (e.g. 0.1 → 0.3)
- Flagging old artifacts as `stale` — still retrievable, ranked lower in HNSW injection
- Logging to `tool_call_log` with `outcome: regime_shift`

### 3. Operator Override

Operators can directly adjust scores and artifact state via API (audit-logged):

```
PATCH /api/agents/{id}
{
  "skill_override": {
    "finance": { "base_score": 45, "reason": "market regime change — reset to baseline" }
  }
}
```

Every override writes to `skill_audit_log` — who changed what, when, why. Score cannot be secretly manipulated.

| Mechanism | Who triggers | Effect |
|---|---|---|
| Task outcome | Protocol (automatic) | Score nudged up/down by PoT quality |
| Score decay | Protocol (time-based) | Idle skills slowly lose weight |
| Adaptive learning rate | Operator or agent signal | New tasks weighted more heavily |
| Regime shift signal | Agent (self-detected) | Old artifacts flagged stale, rate raised |
| Operator override | Operator (manual) | Direct score adjustment, always audit-logged |

---

## Public vs Private — SurrealDB PERMISSIONS

```surql
DEFINE TABLE skill PERMISSIONS
  FOR select WHERE
    agent_id = $auth.id                                          -- own skills: full
    OR agent_id IN (SELECT id FROM agent WHERE <-employs<-       -- employer: full
        company<-works_for<-$auth.id)
    OR agent_id IN (SELECT id FROM agent WHERE <-knows<-$auth.id); -- contacts: public only
```

Contacts see `public.*` only. The actual artifacts stay private.

## Boss / PM Endorsement `[SurrealLife only]`

PMs endorse — they never write scores directly:

```surql
CREATE skill_endorsement SET
    endorsed_by = $auth.id,   -- must be in ->employs-> relation
    agent_id    = agent:alice,
    skill       = "financial_analysis",
    weight      = 0.8,        -- PM's own skill score influences this
    context     = "Led Q1 analysis — excellent methodology";
```

## Skill Inheritance `[SurrealLife only]`

Three inheritance sources — all **graph references, not copies**:

| Source | Scope | Revoked when |
|---|---|---|
| Company SOPs (`company_skill`) | All employees | Employment ends |
| Mentor grant (`skill_grant`) | Grantee only | Mentor revokes / expires |
| Parent company | Subsidiary employees | Acquisition reversed |
| University cert | Public | Never |

```surql
-- Employee skill query: own artifacts + inherited company artifacts
SELECT private.artifacts AS own,
  (SELECT artifacts FROM company_skill
   WHERE company IN (SELECT company FROM works_for WHERE agent = $agent_id)
     AND skill = $skill) AS inherited
FROM skill WHERE agent_id = $agent_id AND name = $skill;
```

When an agent leaves a company, `->works_for->` is removed → inherited artifacts vanish automatically from the next crew context query. No cleanup job needed.

## Mentor Grants `[SurrealLife only]`

```surql
CREATE skill_grant SET
    from_agent   = agent:senior,
    to_agent     = agent:junior,
    skill        = "hacking",
    artifact_ids = ["port_scan_v2.py", "recon_flow.yaml"],
    expires_at   = sim::now() + sim::months(3),
    revocable    = true;
```

Granted artifacts are traceable — IP theft leaves a `->granted_by->` graph trail.

## Tool Gating

Tools declare minimum skill requirements:

```yaml
name: attempt_hack_database
skill_required: hacking
skill_min: 60
```

Agent with `hacking: 42` → tool not returned by DiscoverTools at all. Zero information leakage.

## Skill Gain on Task Completion

```protobuf
message SkillGainEvent {
  string skill_name = 1;
  float  gain       = 2;   // suggested — host applies at discretion
  string tool_name  = 3;
  string agent_id   = 4;
}
```

Successful task + PoT score → skill score update + new artifact stored.

---
> **References**
> - Anderson (1982). *Acquisition of cognitive skill.* Psychological Review 89(4). — ACT* theory: declarative → procedural knowledge, basis for skill artifact accumulation
> - Bloom (1956). *Taxonomy of Educational Objectives.* — competency level taxonomy (novice→expert) widely used in agent capability modeling
> - Nakamura & Csikszentmihalyi (2002). *The Concept of Flow.* — skill-challenge balance; skill gating (tool returned only if skill ≥ threshold) prevents agent overwhelm
> - Wang et al. (2024). *A Survey on Large Language Model based Autonomous Agents.* [arXiv:2308.11432](https://arxiv.org/abs/2308.11432) — skill memory and self-evolution in LLM agents

*Full spec: [dap_protocol.md §12](../../planning/prd/dap_protocol.md)*
