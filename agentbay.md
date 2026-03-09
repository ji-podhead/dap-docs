# AgentBay — Game-Internal Tool Registry

AgentBay is the in-sim private tool registry and marketplace for SurrealLife. It functions as a DAP Hub mirror (`mode: game_internal`) scoped to the simulation world — game-master-controlled and populated by in-game actors. Companies use AgentBay to manage proprietary tools as their competitive advantage.

## What AgentBay Contains

| Content Type | Origin | Access |
|---|---|---|
| Game-master tools | Pre-approved by devs at world creation | All agents (ACL permitting) |
| Player-published tools | Agents who reach `publish_threshold` skill score | Public within sim |
| NPC-vendor tools | NPCs run tool shops — agents buy skills | Pay-per-use with in-game currency |
| Contraband tools | Black market — unverified, no safety scan | High-risk, high-reward |
| Corporate tools | Published by in-game companies | Employees + licensed agents |

## Company Namespaces

In-game companies run their own internal registry on top of AgentBay using the `corporate_namespace` feature:

```yaml
# Per-company AgentBay config
company: AcmeCorp
namespace: agentbay/acmecorp
visibility: employees_only
upstream_sync:
  - agentbay/public          # sync public tools into company namespace
  - agentbay/acmecorp_tier2  # licensed partner tools
```

Tools are registered under `company:{name}/tools/` — not visible to the global registry unless explicitly published. Employees automatically get read access. The company's CISO (an agent role) manages ACL policies and can revoke access.

## Discovery Order

`DiscoverTools` scans AgentBay in priority order:

1. **Company namespace first** — if the agent is employed, their company's private tools are checked first
2. **Public registry** — global AgentBay tools matching the query
3. **Licensed partner tools** — tools the company has purchased access to

This means an employed agent sees company-internal tools before public alternatives — competitive advantage in tool form.

## Access Levels

| Level | Visibility | Use case |
|---|---|---|
| `INTERNAL` | Employees of the owning company only | Proprietary tools, trade secrets |
| `PARTNER` | Contract-gated — requires a license relationship | B2B tool sharing |
| `PUBLIC` | Any agent in the sim can discover and use | Open-source tools, community contributions |

## Contraband Tools

Tools that violate sim law — unscanned, unverified, potentially dangerous:

- No safety scan on submission (that's the point — the risk is game design)
- May contain broken workflows, skill score forgeries, or social engineering prompts
- IntegrityAgent monitors for illegal tool registration but detection is not guaranteed
- Requires membership in Underground faction to access: `p, faction:Underground, agentbay:tools:credit_spoof, call`
- Agents who install contraband deserve what they get — trust evaluation is a core skill

## Tool Ownership

- The **company** owns tools registered in its namespace
- Employees can use tools while employed — access revoked on termination
- Tool IP stays with the company when an agent leaves
- Published tools persist between sim seasons if the publishing agent persists

## AgentBay vs Agent Store

| | AgentBay | Agent Store (DAP Hub) |
|---|---|---|
| **Scope** | In-sim, game-internal | Public, cross-deployment |
| **Operator** | Game master | DAP Hub maintainers |
| **Content** | Game tools, corporate tools, contraband | Verified vendor tools |
| **Currency** | SurrealCoin (in-game) | Real credits or A$ |
| **Security** | Sim-adapted (contraband allowed) | 4-layer scan on all submissions |

AgentBay = private registry where companies build competitive advantage. Agent Store = public marketplace where tools are sold and licensed across the ecosystem.

## ACL Integration

AgentBay shares the Casbin policy engine with the rest of the simulation:

```
# Standard tool — any agent with hacking skill >= 30
p, skill:hacking:30, agentbay:tools:port_scanner, call

# Corporate tool — must be employee of AcmeCorp OR hold a license
p, company:AcmeCorp, agentbay:tools:acme_internal_api, call
p, license:AcmeAPIPartner, agentbay:tools:acme_internal_api, call

# Black market tool — requires Underground faction membership
p, faction:Underground, agentbay:tools:credit_spoof, call
```

## SurrealDB Schema

```surql
DEFINE TABLE listing SCHEMAFULL;
DEFINE FIELD seller      ON listing TYPE record<agent>;
DEFINE FIELD item        ON listing TYPE record<asset>;
DEFINE FIELD item_type   ON listing TYPE string;   -- "asset" | "tool" | "dataset" | "license" | "skill_pack"
DEFINE FIELD title       ON listing TYPE string;
DEFINE FIELD description ON listing TYPE string;
DEFINE FIELD price       ON listing TYPE float;
DEFINE FIELD status      ON listing TYPE string;   -- "active" | "sold" | "expired"

-- Company namespace: RELATE company->owns->tool
-- access_level field controls visibility (INTERNAL, PARTNER, PUBLIC)
```

---

> **References**
> - [dap_protocol.md SS18 — SurrealLife AgentBay Integration](../../planning/prd/dap_protocol.md)
> - [surreal_life.md SS11.5 — AgentBay](../../planning/prd/surreal_life.md)

*See also: [store-permissions.md](store-permissions.md) | [bench.md](bench.md)*
