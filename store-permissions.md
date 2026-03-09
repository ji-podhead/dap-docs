# Agent Store Access Levels — Autonomous Skill Acquisition

Five permission levels control how agents interact with the Agent Store (AgentBay). By default, agents cannot install tools without human approval. Users can grant autonomous store access within defined boundaries — enabling emergent behavior while maintaining control.

## Access Levels

| Level | Discovery | Invocation | Behavior |
|---|---|---|---|
| **NONE** | Not returned by `DiscoverTools` | Blocked | Tool does not exist to the agent. Default for sandboxed agents |
| **READ_ONLY** | Agent can discover and read tool schema | Cannot invoke | Browse-only — agent sees what's available but cannot act |
| **GUARDED** | Full discovery | Invoke allowed, all params logged, result watermarked | Every install queued for human approval |
| **SCOPED** | Full discovery | Invoke allowed within parameter constraints | Autonomous within user-defined boundaries |
| **FULL** | Full discovery | Unrestricted invocation | Maximum autonomy — agent installs anything it can ACL-access |

## How Access Levels Are Set

Access levels are determined by the intersection of:

- **Company policy** — the employing company's security posture
- **Sim law** — simulation-wide rules set by the game master
- **Casbin role** — the agent's ACL role in the policy engine
- **Skill tier** — agents with higher skill scores may unlock higher access

## SCOPED Constraints

`SCOPED` is the recommended production setting. Users define the boundaries:

```yaml
# User-defined agent policy
agent_id: agent_alice
store_access: scoped
constraints:
  max_cost_per_day: 50           # in-game currency budget
  allowed_skill_domains:
    - research
    - writing
    - data_analysis
  blocked_skill_domains:
    - hacking
    - social_engineering
  vendor_tier_minimum: community  # no unverified tools
  require_review_for:
    - tools with skill_min > 60   # senior tools still need approval
    - contraband tools             # always blocked unless explicitly allowed
  notify_on_install: true         # user gets notification even when auto-approved
```

## Upgrade Path

Agents earn higher access through:

| Method | Example |
|---|---|
| **Skill score** | Agent reaches `data_analysis: 60` → unlocks SCOPED for data tools |
| **Endorsement** | Senior agent vouches for the agent's competence |
| **License purchase** | Agent buys a tool license from AgentBay → access granted for that tool |
| **Company promotion** | Agent promoted to senior role → company policy grants FULL access |

## Discovery Integration

At activation, if `store_access >= READ_ONLY`, the `DiscoverTools` response includes:

```
meta_tools:
  - name: browse_store
    description: "Search AgentBay for tools and skill artifacts you can install"
    permission_required: READ_ONLY
  - name: install_from_store
    description: "Install a tool or artifact from AgentBay"
    permission_required: GUARDED
    note: "Will be queued for approval unless you have scoped/full autonomy"
```

## Approval Queue

For `GUARDED` agents, pending installs appear in the oversight dashboard:

```
Agent alice wants to install:
  acmecorp/market-research-suite  [Community Verified]
  Skills gained: research +12, data_analysis +8
  Cost: 15 credits
  Reason: "Need better market data tools for Q3 analysis task"
  [ Approve ]  [ Approve All from this vendor ]  [ Block vendor ]  [ Deny ]
```

Agents can attach a reason to install requests (LLM-generated or structured) — letting users evaluate intent, not just the tool name.

## Skill Economy Enforcement

In SurrealLife, access levels are the enforcement layer for the **skill economy**:

- High-skill tools default to `GUARDED` or `SCOPED` for low-skill agents
- Agents must earn access through skill progression, not just currency
- This prevents unskilled agents from wielding tools they cannot use effectively
- The access level system makes skill investment meaningful — higher skill unlocks better tools

## Skill-Only Installs

Agents can install **skill artifacts only** (no tool code) — knowledge without executable tools:

```yaml
constraints:
  allow_artifact_installs: true    # skill artifacts freely
  allow_tool_installs: false       # no new tool code without approval
```

Useful for users who trust the agent's judgment on knowledge but not on new executable code.

## SurrealDB Implementation

```surql
-- Access level on tool registry
DEFINE FIELD access_level ON tool_registry TYPE string;
-- Values: "NONE" | "READ_ONLY" | "GUARDED" | "SCOPED" | "FULL"

-- PERMISSIONS clause enforces at query time
DEFINE TABLE tool_registry SCHEMAFULL
  PERMISSIONS
    FOR select WHERE access_level IN $auth.access_levels
    FOR update WHERE $auth.role = "admin";
```

---

> **References**
> - [dap_protocol.md SS19 — Agent Store Access Permissions](../../planning/prd/dap_protocol.md)
> - [dap_protocol.md SS18 — AgentBay Integration](../../planning/prd/dap_protocol.md)

*See also: [agentbay.md](agentbay.md) | [teams.md](teams.md)*
