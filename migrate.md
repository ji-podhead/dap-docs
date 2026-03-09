# Migration to DAP — Bringing Existing Tools into DAP

Migrating an existing tool ecosystem to DAP does not require rewriting tools. DAP provides automated conversion utilities, a compatibility bridge, and a phased migration strategy that keeps everything running while tools move over incrementally.

## Migration Paths

| Source Format | Command | What Happens |
|---|---|---|
| MCP tool definitions | `dap-migrate mcp` | JSON schema → DAP YAML, MCP server becomes DAP handler |
| LangChain tools | `dap-migrate langchain` | `BaseTool` subclass → YAML tool + Python builtin handler |
| OpenAI function calls | `dap-migrate openai-functions` | JSON schema → DAP YAML |
| Plain Python functions | `dap-migrate python` | Introspects type hints + docstrings → DAP YAML |
| YAML function definitions | `dap-migrate yaml` | Common agent YAML formats → DAP YAML |

### From MCP

MCP dumps all tool descriptions into the system prompt at session start. DAP replaces this with `DiscoverTools` — tools are loaded on demand within a token budget. Migration adds `bloat_score` (token efficiency) and `skill_required` (access gating) to each tool definition.

### From LangChain

Replace `@tool` decorators with YAML registration. Tools become discoverable via Qdrant vector search instead of hardcoded in the agent graph. LangChain memory stores migrate to SurrealDB HNSW for unified vector search.

### From AutoGen

Agent conversations map to DAP `InvokeTool` + MQTT inbox messaging. Shared memory between AutoGen agents becomes SurrealDB graph relationships (`RELATE agent->knows->agent`).

### From OpenAI Functions

JSON schema definitions map directly to DAP YAML handler definitions. `function_call` becomes `InvokeTool` gRPC. Response parsing stays the same — DAP returns structured results.

### From Plain Python

Wrap functions in handler YAML and get ACL, skill gating, and audit logging for free. `dap-migrate python` introspects type hints and docstrings to generate the YAML automatically.

## Migration CLI

```bash
# Install
pip install dap-migrate

# Convert a directory of MCP tools (dry run first)
dap-migrate mcp ./mcp-tools/ --output ./dap-tools/ --dry-run

# Convert with ACL defaults and auto skill gating
dap-migrate mcp ./mcp-tools/ \
  --output ./dap-tools/ \
  --default-acl "role:agent, call" \
  --skill-gating auto   # infer skill requirements from tool descriptions using LLM

# Convert LangChain toolkits
dap-migrate langchain myapp.tools:MyToolkit --output ./dap-tools/

# Register converted tools to a running DAP server
dap-migrate register ./dap-tools/ --server grpc://localhost:50051 --admin-key $DAP_ADMIN_KEY
```

`--skill-gating auto` uses an LLM to infer `skill_min` and `skill_required` fields from tool descriptions. Optional — set manually after conversion.

## MCP Compatibility Bridge

For teams running MCP and DAP side by side during migration:

```yaml
# dap-server config
mcp_bridge:
  enabled: true
  mcp_server_url: "stdio://./my-mcp-server"  # or HTTP
  expose_as_dap_tools: true    # MCP tools appear in DiscoverTools results
  acl_passthrough: false       # apply DAP ACL to bridged MCP tools
  namespace: "mcp"             # tools appear as mcp/tool_name
```

Bridged MCP tools are indistinguishable from native DAP tools. ACL, skill gating, and audit logging apply at the bridge layer. Use the `a2a://` prefix to wrap existing MCP/OpenAI tools as DAP tools.

## Phased Migration Strategy

```
Phase 1 — Bridge
  Enable mcp_bridge
  All MCP tools appear in DAP discovery
  Agents use DAP for discovery, MCP bridge for execution
  No tool rewrites needed

Phase 2 — Convert high-value tools
  Run dap-migrate on priority tools
  Register native DAP versions alongside bridge
  Native DAP tool takes precedence (higher confidence in Qdrant)
  Bridge still handles remaining tools

Phase 3 — Retire bridge
  All tools converted
  Disable mcp_bridge
  Full native DAP
```

## Feature Comparison After Migration

| Feature | MCP (before) | DAP native (after) |
|---|---|---|
| ACL-gated visibility | No | Yes — per-agent |
| Skill gating | No | Yes — configurable |
| Semantic discovery | No (name-only) | Yes — Qdrant vector search |
| Artifact binding | No | Yes — inject workflow artifacts |
| Streaming responses | Limited | First-class via gRPC streaming |
| Audit log | Per-server (if impl.) | Centralized in SurrealDB |
| Multi-tenant isolation | No | Yes — DAP Teams namespacing |
| Version management | No | Yes — tool versions in registry |

## Quick-Start Checklist

1. `pip install dap-migrate` — install the migration CLI
2. `dap-migrate mcp ./tools/ --output ./dap-tools/ --dry-run` — preview conversion
3. Review generated YAML, add `skill_required` and `skill_min` where appropriate
4. `dap-migrate register ./dap-tools/ --server grpc://localhost:50051` — register tools
5. Verify with `dap-cli discover --query "your tool"` — confirm tools appear in discovery

Migration is complete when no tool names appear hardcoded in agent prompt templates. Tools are discovered, not listed.

---

> **References**
> - [dap_protocol.md SS20 — DAP Migration](../../planning/prd/dap_protocol.md)
> - MCP Specification: [modelcontextprotocol.io](https://modelcontextprotocol.io)
> - LangChain Tools: [python.langchain.com/docs/how_to/custom_tools](https://python.langchain.com/docs/how_to/custom_tools)

*Full spec: [dap_protocol.md SS20](../../planning/prd/dap_protocol.md)*
