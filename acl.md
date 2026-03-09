# DAP ACL — Three-Layer Stack Reference

DAP uses a three-layer access control architecture. Each layer covers a distinct enforcement surface — no single layer can replace the others.

## Layer 1: Casbin — Protocol & Application ACL

Casbin with `keyMatch2` path wildcards enforces access at the protocol level. The same policy store covers gRPC tool calls, MQTT topics, physical rooms, and data namespaces.

### Tool ACL Examples

```
p, role:agent,        /tools/send_message,           call
p, role:agent,        /tools/http_request,           call
p, role:ceo,          /tools/fire_agent,             call
p, role:hacker_tier2, /tools/attempt_hack/web,       call
p, role:hacker_tier4, /tools/attempt_hack/database,  call
p, role:referee,      /tools/rag_query/:any,         call
p, lic:lawyer,        /tools/file_lawsuit,           call
p, lic:medical,       /tools/diagnose,               call
p, game_master,       /tools/*,                      call
```

### MQTT Topic ACL (Same Store)

```
p, role:agent,    dap/agents/+/inbox,       subscribe
p, role:agent,    dap/agents/$self/outbox,  publish
p, role:ceo,      dap/company/+/broadcast,  subscribe
p, game_master,   dap/#,                    all
```

### Forbidden Tools

Globally denied — no policy can grant access:

```
deny, *, /tools/audit_log_delete, call
deny, *, /tools/agent_identity_transfer, call
```

All ACL checks run **before** handler execution. If denied, `ToolError(permission_denied)` returns immediately.

## Layer 2: SurrealDB RBAC — Row-Level Data Security

SurrealDB's `PERMISSIONS FOR select WHERE` clauses and `$auth` JWT parameters filter **which records** an agent can read. This operates entirely inside SurrealQL.

```surql
-- Tool registry: agents see tools in their tier or below
DEFINE TABLE tool_registry PERMISSIONS
  FOR select WHERE $auth.skill_tier >= tier OR $auth.role = 'game_master'
  FOR create, update, delete WHERE $auth.role = 'game_master';

-- Audit log: agents see only their own invocations
DEFINE TABLE dap_audit PERMISSIONS
  FOR select WHERE agent_id = $auth.id OR $auth.role IN ['game_master', 'referee'];

-- Agent memory: private to owner
DEFINE TABLE agent_memory PERMISSIONS
  FOR select WHERE owner_id = $auth.id;
```

Authentication via SurrealDB Record Users:

```surql
DEFINE ACCESS agent ON DATABASE TYPE RECORD
  SIGNUP (CREATE agent SET name = $name, role = $role, skill_tier = 0)
  SIGNIN (SELECT * FROM agent WHERE id = $id AND token = $token);
```

## Layer 3: SurrealDB Capabilities — Query Surface Hardening

`--deny-arbitrary-query=record` restricts what queries agents can send, independent of RBAC row filtering.

### Production DAPNet Config

```bash
surreal start \
  --deny-all \
  --allow-funcs "array,string,math,vector,time,crypto::argon2,http::post,http::get" \
  --allow-net "mqtt-broker:1883,dap-grpc:50051,generativelanguage.googleapis.com:443" \
  --deny-arbitrary-query "record,guest" \
  --deny-scripting
```

Agents (Record Users) cannot send raw SurrealQL. They use `DEFINE API` endpoints only:

```surql
DEFINE API /agent/graph/contacts METHOD GET
  PERMISSIONS WHERE $auth.role IN ["agent","ceo"]
  THEN {
    SELECT ->knows->agent.{id, name, expertise, skill_tier}
    FROM $auth.id
  };

DEFINE API /agent/memory/search METHOD POST
  PERMISSIONS WHERE $auth.role = "agent"
  THEN {
    SELECT id, context, outcome, pnl,
           vector::similarity::cosine(embedding, $body.query_vec) AS score
    FROM trade_experience
    WHERE agent_id = $auth.id
    ORDER BY score DESC LIMIT 5
  };
```

`--allow-net` scoped to DAPNet-internal services only — agents cannot reach arbitrary external URLs via DB functions.

## Why Neither Layer Works Alone

| Enforcement Target | SurrealDB RBAC | Casbin |
|---|---|---|
| DB record row visibility | Native (`PERMISSIONS FOR select WHERE`) | No DB row access |
| gRPC `InvokeTool` permission | Runs before DB query | Policy path check |
| MQTT topic subscribe/publish | Not involved | Topic ACL policies |
| Wildcard path matching (`/tools/hack/*`) | No concept of paths | `keyMatch2` native |
| Dynamic runtime policy updates | Schema change required | Hot reload |
| Cross-resource unified policy | Per-table only | One policy for rooms + tools + topics |

**Example:** Agent calls `InvokeTool("attempt_hack/web")` → Casbin checks `role:agent` against `/tools/attempt_hack/web` (denied) **before** any DB query runs. SurrealDB RBAC would never see the request. Conversely, SurrealDB RBAC filters `SELECT * FROM agent_memory` at the query level — Casbin cannot do row-level filtering.

## Hybrid Pattern — Identity Flow

```
1. Agent authenticates via SurrealDB → gets JWT ($auth.role, $auth.skill_tier)
2. DAP server extracts identity from JWT
3. Casbin enforces protocol-level access (gRPC paths, MQTT topics)
4. Tool executes — SurrealDB PERMISSIONS filter DB reads automatically
```

```python
token = surreal.signin(agent_id=agent_id, token=agent_token)
subject = f"role:{token['role']},lic:{token['license']}"

if not enforcer.enforce(subject, f"/tools/{tool_name}", "call"):
    raise PermissionDenied

result = tool.execute(params, db_session=surreal_session)
```

One identity source (SurrealDB JWT) feeds both layers — no duplicate user management.

## Three-Layer Diagram

```
Request: agent calls SurrealDB RPC query()
  │
  ├─ Layer 3: Capabilities
  │   --deny-arbitrary-query=record → blocked unless DEFINE API endpoint
  │   --allow-funcs whitelist → no http::* to unlisted targets
  │
  ├─ Layer 2: SurrealDB RBAC
  │   PERMISSIONS FOR select WHERE $auth.role = ... → row-level filtering
  │   DEFINE API PERMISSIONS WHERE ... → endpoint-level check
  │
  └─ Layer 1: Casbin (gRPC InvokeTool path)
      role:agent /tools/attempt_hack/web call → denied
      (separate transport, same identity)
```

> **References**
> - Casbin: [An Authorization Library that Supports Access Control Models](https://casbin.org/docs/overview)
> - NIST RBAC Model: [Role Based Access Control (ANSI INCITS 359-2004)](https://csrc.nist.gov/pubs/conference/2000/10/13/proceedings-of-the-5th-acm-sacmat/final)
> - SurrealDB RBAC: [SurrealDB Access Control](https://surrealdb.com/docs/surrealdb/security/authentication)

*Full spec: [dap_protocol.md §8](../../planning/prd/dap_protocol.md)*
