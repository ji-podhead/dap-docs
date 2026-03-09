# DAP Teams — Multi-Tenant Deployment

DAP Teams is the multi-tenant deployment model for DAP. Each tenant gets an isolated namespace with its own tool registry, ACL policies, and skill profiles — suitable for SaaS platforms, research labs, and enterprises running multiple agent fleets on shared infrastructure.

## Tenant Isolation

Each tenant gets a logical namespace. Data never crosses tenant boundaries:

```
/tenant/{tenant_id}/tools/*     ← tool registry partition
/tenant/{tenant_id}/acl/*       ← casbin policy partition
/tenant/{tenant_id}/skills/*    ← skill profile partition
```

A `DiscoverTools` call always operates within the caller's tenant namespace. Tool registration, ACL policies, and skill data are fully partitioned. SurrealDB provides the namespace isolation; Casbin provides the policy isolation.

## Tenant Management API

```
POST   /admin/tenants                    → create tenant
DELETE /admin/tenants/{tenant_id}        → delete tenant + all data
POST   /admin/tenants/{tenant_id}/tools  → register tool for tenant
GET    /admin/tenants/{tenant_id}/tools  → list tools in tenant
POST   /admin/tenants/{tenant_id}/acl    → add casbin policy for tenant
GET    /admin/tenants/{tenant_id}/audit  → view tool call log for tenant
```

## Team Tiers

| Tier | Agents | DAPNet | Features |
|---|---|---|---|
| **Free** | 3 | Shared MQTT namespace | Basic discovery + invocation |
| **Pro** | 10 | Dedicated MQTT namespace | Priority routing, private channels |
| **Enterprise** | Unlimited | Dedicated infrastructure | Custom SLAs, private Hub mirror |

### Agent Quota Management

When a team hits its agent quota, new agent registrations are rejected. Mitigation: **crews** — one agent coordinates multiple sub-agents as a single registered entity, keeping within quota while scaling capability. Crews are the efficiency mechanism, not a workaround.

## Cross-Team Tool Sharing

Tools are private to their tenant by default. Publishing options:

| Visibility | Who can discover |
|---|---|
| `tenant-private` | Only agents in the same tenant (default) |
| `team-public` | Agents in any tenant on the same DAP Teams instance |
| `platform-public` | Published to DAP Hub — discoverable by any DAP deployment |

## Billing

DAP Teams meters usage at the tenant level:

| Metric | Description |
|---|---|
| **Per invocation** | A$ or credits per `InvokeTool` call |
| **Per discovery** | Count of `DiscoverTools` calls |
| **Per registration** | One-time cost for registering a new tool |
| **Compute time** | Billed per handler execution second |

PoD (Proof of Delivery) certificates serve as the authoritative invocation count for billing. Every tool call produces a signed PoD — the billing system counts these, not internal logs.

## Human-in-the-Loop

Humans are first-class participants in DAP Teams — not just observers:

- **Approval gates**: tool invocations can require human sign-off before execution
- **Async input**: agents request human input via the approval queue, continue other work while waiting
- **Slack bot integration**: approval requests and notifications delivered to team Slack channels
- **Dashboard oversight**: pending approvals, invocation history, and cost tracking in the admin UI

## DAPNet Cross-Team Visibility

Teams on the same DAP Teams instance can subscribe to shared MQTT topics for coordination:

```
dap/teams/{team_id}/announcements   # team-wide broadcasts
dap/teams/public/events             # cross-team event stream
```

MQTT topic subscriptions replace sync meetings for cross-team coordination. Full detail on DAPNet messaging in [dapnet.md](dapnet.md).

## University Onboarding

New agents joining a team face a cold-start problem — no skill history, no tool familiarity. **Fast-track courses** fix this:

- Pre-configured skill artifact bundles installed on agent registration
- Curated tool discovery sets — new agents see essential tools first
- Onboarding workflows that walk agents through team-specific tools and conventions

Universities are the team's investment in agent quality — agents who complete onboarding are productive faster.

---

> **References**
> - [dap_protocol.md SS14 — DAP Teams](../../planning/prd/dap_protocol.md)
> - [dap_protocol.md SS15 — DAP Hub](../../planning/prd/dap_protocol.md)

*See also: [dapnet.md](dapnet.md) | [store-permissions.md](store-permissions.md)*
