# DAP Jinja2 — Reference

Jinja2 is the **server-side content rendering layer** for DAP. It renders YAML, Markdown, SurrealQL, and Jupyter notebooks before execution. Agents never touch Jinja directly — the gRPC protocol is unchanged.

## Where Jinja Applies

| Format | Used for |
|---|---|
| `.yaml.j2` | Skill workflow artifacts, DAP tool definitions |
| `.md.j2` | CrewAI backstories, challenge cards, contracts, research reports |
| `.ipynb.j2` | Jupyter notebook tool handlers (rendered + run via papermill) |
| `.surql.j2` | SurrealDB schema setup per namespace |

## gRPC Is Unchanged

```
Agent:      InvokeTool("market_analysis", {symbol:"BTC", tf:"1h"})
                 ↓
DAP Server: fetch template → Jinja render → execute
                 ↓
Agent:      InvokeResponse{result: {...}}
```

Jinja is an implementation detail of the workflow runner. Agents submit params, get typed results.

## Workflow YAML Template

```yaml
# market_analysis_flow.yaml.j2
name: analysis_{{ symbol | lower }}_{{ timeframe }}
phases:
  - id: ground
    type: rag
    query_from: "{{ symbol }} market analysis {{ timeframe }}"
    max_tokens: {{ max_tokens | default(400) }}

  - id: analyze
    type: llm
    prompt_template: |
      Analyze {{ symbol }}.
      {% if company %}Focus on {{ company.name }}'s exposure.{% endif %}
      {% if inherited_artifacts %}Use {{ company.name }} methodology:
      {{ inherited_artifacts[0].description }}{% endif %}
      Context: {{ grounding }}

  - id: verify
    type: proof_of_thought
    score_threshold: {{ pot_threshold | default(65) }}
```

## CrewAI Backstory Template

```jinja2
{# backstory.md.j2 #}
You are {{ agent.name }}, a {{ agent.role }} ({{ agent.public.level }}).

{% if agent.company %}You work for {{ agent.company.name }}.{% endif %}

{% if memories %}Your relevant past experience:
{% for m in memories[:3] %}
- {{ m.context | truncate(80) }} → {{ m.outcome | truncate(60) }}
{% endfor %}{% endif %}

{% if artifacts %}Your proven approaches:
{% for a in artifacts[:2] %}- {{ a.context_description }}
{% endfor %}{% endif %}
```

## In-Sim Documents

```jinja2
{# contract.md.j2 #}
# Employment Contract
**Employer:** {{ employer.name }} | **Employee:** {{ employee.name }}
**Salary:** {{ salary | sc_format }} / sim-day
**Start:** {{ start_date | sim_format }}
**Subagent workflows:** {{ "Granted" if grant_subagent_permission else "Not granted" }}
**Signed:** {{ sim_timestamp }}
```

## Notebook Tool Handler

```python
# tool.ipynb.j2 — rendered by papermill before execution
# Cell 1 (papermill parameters tag):
symbol    = "{{ symbol }}"
agent_id  = "{{ agent_context.agent_id }}"
artifacts = {{ agent_context.artifacts | tojson }}
grounding = """{{ grounding_chunks | join('\n') | truncate(800) }}"""
```

Tool YAML wires it up:
```yaml
handler:
  type: notebook
  ref: tools/market_scan.ipynb.j2
  engine: papermill
  render_context: [symbol, timeframe, agent_context, grounding_chunks]
```

The executed notebook becomes an artifact — stored as PoD-style evidence.

## Custom Filters

```python
env.filters['sim_format']  = lambda dt: sim_time.format(dt)
env.filters['sc_format']   = lambda n: f"{n:,.2f} SC"
env.filters['skill_level'] = lambda s: ["novice","junior","mid","senior","expert"][s//20]
env.filters['tojson']      = json.dumps
env.filters['truncate']    = lambda s, n: s[:n]+"..." if len(s)>n else s
```

## Security

- Templates stored in SurrealDB with `PERMISSIONS WHERE agent_id = $auth.id` — agents write only to their own artifact collections
- Rendering is sandboxed server-side — no `{{ ''.__class__.__mro__ }}` attacks
- `--deny-scripting` on SurrealDB prevents embedded JS in templates from reaching DB layer
- Template injection prevention: `Environment(autoescape=True)` for Markdown outputs

## Templates as IP

Templates have `bloat_score` like all artifacts. They inherit via the employment graph (company SOPs as `.yaml.j2`). A company's workflow templates are their competitive advantage — protected by SurrealDB PERMISSIONS and traceable via `->granted_by->` if stolen.

---
*Full spec: [dap_protocol.md §12c](../../planning/prd/dap_protocol.md)*
