# DAP Tool Registration — Reference

Tools in DAP are registered into a Qdrant vector index backed by SurrealDB records. Registration is the entry point for any tool — built-in or custom — to become discoverable and invocable.

## YAML Tool Definition

Every tool is defined as a YAML file with a standard structure:

```yaml
name: check_company_balance
description: "Returns the current A$ balance for a company"
version: "1.0.2"
parameters:
  company_id:
    type: string
    required: true
    description: "SurrealDB record ID, e.g. company:alphastack"
acl_path:        /tools/check_company_balance
acl_action:      call
allowed_roles:   [agent, ceo, referee]
skill_required:  financial_analysis
skill_min:       0                      # 0 = no minimum
handler:
  type: surreal_query
  query: "SELECT balance FROM company WHERE id = $company_id"
  return_field: balance
skill_linked:    financial_analysis
skill_gain:      0.1
a2a:             false                  # true = auto-generates Agent Card
bloat_score:                            # computed at registration, can be overridden
  description_tokens: 18
  schema_tokens: 94
  artifact_tokens: 0
  total: 112
```

### Key Fields

| Field | Required | Description |
|---|---|---|
| `name` | yes | Unique tool identifier |
| `description` | yes | One sentence — what the tool does |
| `version` | no | Semver string (e.g. `1.0.2`) |
| `parameters` | yes | JSON Schema-compatible parameter definitions |
| `acl_path` | yes | Casbin path for access control |
| `allowed_roles` | yes | Roles that can call this tool |
| `skill_required` | no | Skill name required to use this tool |
| `skill_min` | no | Minimum skill score (0 = no minimum) |
| `handler` | yes | Handler configuration (see below) |
| `a2a` | no | If `true`, auto-generates an A2A Agent Card for cross-agent exposure |
| `bloat_score` | auto | Computed at registration time (see [bloat-score.md](bloat-score.md)) |

## Handler Types

| Type | Description | Execution |
|---|---|---|
| `builtin` | Python functions registered at GTS startup | Direct, no sandbox |
| `yaml_template` | Declarative YAML — parameter substitution + DB query | GTS template engine |
| `notebook` | `.ipynb` cells in sandboxed subprocess | Isolated, no network, read-only DB |
| `proof` | Proof of Search pipeline — Z3-verified claims | Streamed, Referee-controlled |
| `a2a` | Delegates to another agent via A2A protocol | Cross-agent RPC |
| `subagent` | Spawns a sub-agent for the task | LangGraph sub-activation |
| `crew` | CrewAI multi-agent crew execution | CrewAI runtime |

### YAML Template Example

```yaml
handler:
  type: surreal_query
  query: "SELECT balance FROM company WHERE id = $company_id"
  return_field: balance
```

No code, no deploy — file drop into `/surreal_config/tools/custom/`.

### Notebook Example

Custom Python logic, sandboxed per invocation. No persistent state, no network, read-only DB. Timeout configurable (default: 5s).

### Proof Handler

Wraps Proof of Search as a tool type. Agent submits a thesis, Referee controls sandboxed search, Z3 verifies the conclusion was derived from evidence.

## Registration Flow

```
1. Tool YAML submitted (file drop, admin API, or agent-authored)
     │
2. Safety scan (agent-authored tools only):
     ├─ Sandbox execution (isolated, no network, no DB write)
     ├─ Static analysis (ACL path references, external API calls)
     └─ IntegrityAgent review flag (sensitive categories)
     │
3. bloat_score computed:
     ├─ description_tokens = tokenize(name + description)
     ├─ schema_tokens = tokenize(parameter_schema + return_schema)
     ├─ artifact_tokens = tokenize(bound artifacts, if any)
     └─ total = sum, grade assigned (A/B/C/D)
     │
4. Qdrant indexed:
     └─ vector = embed("{name} {description} {tags}")
     │
5. SurrealDB record created in tool_registry
     │
6. index_version bumped → all active agents re-discover on next activation
```

## Who Can Register

| Source | Mechanism | Review |
|---|---|---|
| Game masters | Drop YAML into `/surreal_config/tools/custom/` | Auto-registered |
| Agents (approved) | Write code → safety scan → `register_tool` admin API | Game master review optional |
| Platform | Built-in tools registered at GTS startup | None |

## Tool Versioning

Use semver in the `version` field. When a tool is updated:
- New version is registered alongside the old
- `deprecated: true` flag on the old version
- `index_version` bumps → agents re-discover and see the updated tool
- Old versions remain callable until explicitly removed

## A2A Exposure

Setting `a2a: true` on a tool auto-generates an A2A Agent Card, making the tool discoverable by agents on other DAPNet nodes. The card includes the tool's name, description, parameters, and ACL requirements.

## SurrealDB Event-Driven Rediscovery

```surql
DEFINE EVENT tool_change ON TABLE tool_registry
  WHEN $event = "CREATE" OR $event = "UPDATE"
  THEN {
    -- bump index_version, notify connected agents
    UPDATE dap_meta:index SET version = time::now();
    http::post("http://dap-grpc:50051/notify", { event: "tool_change" });
  };
```

When a tool is created or modified, the event triggers `index_version` update and agent rediscovery — no restart, no manual intervention.

> **References**
> - [Qdrant HNSW Index](https://qdrant.tech/documentation/concepts/indexing/)
> - [SurrealDB Events](https://surrealdb.com/docs/surrealdb/surrealql/statements/define/event)

*Full spec: [dap_protocol.md §4, §5, §9](../../planning/prd/dap_protocol.md)*
