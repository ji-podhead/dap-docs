# DAP Client — Reference

DAP's API contract is defined in protobuf. The `.proto` file is the SDK — generate stubs in any gRPC-supported language and connect. No wrapper library required.

---

## Proto File

The canonical source:

```
dap/proto/tool_service.proto
```

Key service definition (see [protocol.md](protocol.md) for full spec):

```protobuf
syntax = "proto3";
package dap.v1;

service ToolService {
  rpc DiscoverTools (DiscoverRequest)  returns (DiscoverResponse);
  rpc SearchTools   (SearchRequest)    returns (SearchResponse);
  rpc GetToolSchema (SchemaRequest)    returns (ToolSchema);
  rpc InvokeTool    (InvokeRequest)    returns (stream InvokeResponse);
}

message DiscoverRequest {
  string agent_id  = 1;
  string context   = 2;
  int32  max_tools = 3;
}

message InvokeRequest {
  string agent_id    = 1;
  string tool_name   = 2;
  string params_json = 3;   // JSON-encoded tool parameters
}

message InvokeResponse {
  string chunk      = 1;    // streaming result chunks
  bool   is_final   = 2;
  string result_json = 3;   // set on final chunk
  string error      = 4;
}
```

---

## Python Client

### Install

```bash
pip install grpcio grpcio-tools
```

### Generate stubs

```bash
python -m grpc_tools.protoc \
  -I ./proto \
  --python_out=./dap_client \
  --grpc_python_out=./dap_client \
  ./proto/tool_service.proto
```

### Connect + auth

Agent tokens are passed as gRPC metadata on every call:

```python
import grpc
from dap_client import tool_service_pb2 as pb
from dap_client import tool_service_pb2_grpc as stub

AGENT_TOKEN = "your-agent-token"
DAP_SERVER  = "dap.yourdeployment.com:50051"

# Reuse channel across calls — connection is cheap, channel is not
channel = grpc.secure_channel(DAP_SERVER, grpc.ssl_channel_credentials())
client  = stub.ToolServiceStub(channel)

def _meta():
    return [("authorization", f"Bearer {AGENT_TOKEN}")]
```

### DiscoverTools

```python
def discover(context: str, max_tools: int = 20) -> list[pb.ToolSummary]:
    req  = pb.DiscoverRequest(agent_id="my-agent", context=context, max_tools=max_tools)
    resp = client.DiscoverTools(req, metadata=_meta())
    return list(resp.tools)

tools = discover("analyze BTC market over 4h timeframe")
for t in tools:
    print(t.name, t.description)
```

### InvokeTool (streaming)

```python
import json

def invoke(tool_name: str, params: dict) -> str:
    req = pb.InvokeRequest(
        agent_id   = "my-agent",
        tool_name  = tool_name,
        params_json= json.dumps(params),
    )
    result = ""
    for chunk in client.InvokeTool(req, metadata=_meta()):
        if chunk.error:
            raise RuntimeError(chunk.error)
        if chunk.is_final:
            result = chunk.result_json
    return result

output = invoke("market_analysis", {"symbol": "BTC", "timeframe": "4h"})
print(json.loads(output))
```

### Full example: discover → invoke

```python
# 1. Discover tools for the current task
tools = discover("portfolio risk calculation")

# 2. Pick the right tool (your agent's LLM does this in practice)
tool = next(t for t in tools if "risk" in t.name)

# 3. Fetch schema if needed
schema_req  = pb.SchemaRequest(tool_name=tool.name)
schema_resp = client.GetToolSchema(schema_req, metadata=_meta())
print(schema_resp.params_schema_json)

# 4. Invoke
result = invoke(tool.name, {"portfolio_id": "p-123", "confidence": 0.95})
```

---

## TypeScript / Node.js Client

### Install

```bash
npm install @grpc/grpc-js @grpc/proto-loader
# For stub generation with protoc:
npm install -g grpc-tools ts-proto
```

### Generate stubs (ts-proto)

```bash
protoc \
  --plugin=./node_modules/.bin/protoc-gen-ts_proto \
  --ts_proto_out=./src/dap_client \
  --ts_proto_opt=outputServices=grpc-js \
  -I ./proto \
  ./proto/tool_service.proto
```

This generates `tool_service.ts` with typed request/response interfaces and a `ToolServiceClient` class.

### Connect + auth

```typescript
import * as grpc from "@grpc/grpc-js";
import { ToolServiceClient } from "./dap_client/tool_service";

const AGENT_TOKEN = process.env.DAP_AGENT_TOKEN!;
const DAP_SERVER  = "dap.yourdeployment.com:50051";

const client = new ToolServiceClient(
  DAP_SERVER,
  grpc.credentials.createSsl()
);

const meta = () => {
  const m = new grpc.Metadata();
  m.set("authorization", `Bearer ${AGENT_TOKEN}`);
  return m;
};
```

### DiscoverTools

```typescript
import { DiscoverRequest } from "./dap_client/tool_service";

async function discover(context: string, maxTools = 20) {
  return new Promise<ToolSummary[]>((resolve, reject) => {
    const req: DiscoverRequest = {
      agentId:  "my-agent",
      context,
      maxTools,
    };
    client.discoverTools(req, meta(), (err, resp) => {
      if (err) return reject(err);
      resolve(resp!.tools);
    });
  });
}

const tools = await discover("analyze BTC market over 4h timeframe");
tools.forEach(t => console.log(t.name, t.description));
```

### InvokeTool (streaming)

```typescript
import { InvokeRequest } from "./dap_client/tool_service";

async function invoke(toolName: string, params: object): Promise<string> {
  return new Promise((resolve, reject) => {
    const req: InvokeRequest = {
      agentId:    "my-agent",
      toolName,
      paramsJson: JSON.stringify(params),
    };

    const stream = client.invokeTool(req, meta());
    let result = "";

    stream.on("data", chunk => {
      if (chunk.error) return reject(new Error(chunk.error));
      if (chunk.isFinal) result = chunk.resultJson;
    });

    stream.on("end",   () => resolve(result));
    stream.on("error", reject);
  });
}

const output = await invoke("market_analysis", { symbol: "BTC", timeframe: "4h" });
console.log(JSON.parse(output));
```

### Full example: discover → invoke

```typescript
// 1. Discover
const tools = await discover("portfolio risk calculation");

// 2. Pick tool
const tool = tools.find(t => t.name.includes("risk"))!;

// 3. Invoke
const result = await invoke(tool.name, { portfolioId: "p-123", confidence: 0.95 });
```

---

## JavaScript (CommonJS / no types)

If you prefer raw `@grpc/proto-loader` without code generation:

```js
const grpc      = require("@grpc/grpc-js");
const protoLoad = require("@grpc/proto-loader");

const AGENT_TOKEN = process.env.DAP_AGENT_TOKEN;
const DAP_SERVER  = "dap.yourdeployment.com:50051";

const pkgDef = protoLoad.loadSync("./proto/tool_service.proto", {
  keepCase:     true,
  longs:        String,
  enums:        String,
  defaults:     true,
  oneofs:       true,
});
const { dap: { v1: { ToolService } } } = grpc.loadPackageDefinition(pkgDef);

const client = new ToolService(DAP_SERVER, grpc.credentials.createSsl());

function meta() {
  const m = new grpc.Metadata();
  m.set("authorization", `Bearer ${AGENT_TOKEN}`);
  return m;
}

// DiscoverTools
client.DiscoverTools(
  { agent_id: "my-agent", context: "market analysis", max_tools: 20 },
  meta(),
  (err, resp) => {
    if (err) throw err;
    resp.tools.forEach(t => console.log(t.name));
  }
);

// InvokeTool (streaming)
const call = client.InvokeTool(
  { agent_id: "my-agent", tool_name: "market_analysis", params_json: JSON.stringify({ symbol: "BTC" }) },
  meta()
);
call.on("data",  chunk => { if (chunk.is_final) console.log(chunk.result_json); });
call.on("error", err   => console.error(err));
```

---

## MQTT Client (DAP Messaging)

For event subscriptions and agent-to-agent messaging. See [messaging.md](messaging.md) for full topic schema.

### Python (paho-mqtt)

```python
import paho.mqtt.client as mqtt
import json

MQTT_HOST  = "mqtt.yourdeployment.com"
AGENT_ID   = "my-agent"

def on_message(client, userdata, msg):
    payload = json.loads(msg.payload)
    print(f"[{msg.topic}] {payload}")

mqttc = mqtt.Client(client_id=AGENT_ID, protocol=mqtt.MQTTv5)
mqttc.username_pw_set(AGENT_ID, password="your-mqtt-token")
mqttc.tls_set()
mqttc.on_message = on_message

mqttc.connect(MQTT_HOST, 8883)

# Subscribe to agent inbox
mqttc.subscribe(f"agent/{AGENT_ID}/inbox", qos=1)

# Subscribe to tool call logs for your agent
mqttc.subscribe(f"logs/tool_calls/{AGENT_ID}/#", qos=0)

mqttc.loop_forever()
```

### JavaScript (mqtt.js)

```js
const mqtt = require("mqtt");

const AGENT_ID = "my-agent";

const client = mqtt.connect("mqtts://mqtt.yourdeployment.com:8883", {
  clientId: AGENT_ID,
  username: AGENT_ID,
  password: process.env.MQTT_TOKEN,
});

client.on("connect", () => {
  client.subscribe(`agent/${AGENT_ID}/inbox`,       { qos: 1 });
  client.subscribe(`logs/tool_calls/${AGENT_ID}/#`, { qos: 0 });
});

client.on("message", (topic, payload) => {
  console.log(topic, JSON.parse(payload.toString()));
});
```

---

## Auth Summary

| Method | Where | Value |
|---|---|---|
| gRPC | `authorization` metadata header | `Bearer <agent_token>` |
| MQTT | username + password | agent_id + mqtt_token |
| REST (DAP Apps) | `Authorization` header | `Bearer <agent_token>` |

Tokens are issued per agent by the DAP server. Rotate via `POST /agents/{id}/rotate-token`.

---

*See also: [protocol.md](protocol.md) · [messaging.md](messaging.md) · [acl.md](acl.md) · [apps.md](apps.md)*
