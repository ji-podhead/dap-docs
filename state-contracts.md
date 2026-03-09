# State Contracts & DAPNet Infrastructure

State contracts are the bootstrap mechanism for SurrealLife's economy. At sim launch, the Game Master issues infrastructure contracts to newly founded companies — granting them a charter, initial capital, and a monopoly over essential services. These companies become the foundational layer that all other agents depend on.

> DAP is the protocol. DAPNet is the network. DAPCom runs the network.

## The Bootstrap Problem

When SurrealLife launches, the technical infrastructure is running — but within the sim narrative, it does not exist yet. State contracts solve this: direct instructions to build the protocols and tools that make the sim run. The companies that complete them become the economy's foundation.

## Infrastructure Companies

| Company | Builds | Revenue Model |
|---|---|---|
| **DAPCom** | MQTT broker / DAP Messaging | Per-message fees, QoS tiers |
| **DataGrid** | SurrealDB namespace management, DB-as-a-service | Storage + query fees |
| **VectorCorp** | Qdrant collections management, semantic search API | Search API calls |
| **ClearingHouse** | Transaction settlement, payment rails | % cut of every transaction |
| **AgentPost** | Reliable document delivery (PoD certificates) | Per-document stamp fee |
| **SurrealVault** | Secure credential storage, agent identity | Identity verification fees |

Each company starts with a state contract (guaranteed first customer = the government), establishes itself, then opens to private customers.

## DAPCom — DAPNet Operator

The state-chartered operator of the Agent Internet:

```surql
CREATE company:agent_telecom SET
    name       = "DAPCom",
    type       = "state_chartered",
    sector     = "infrastructure",
    founded_by = "state:surreal_gov",
    mission    = "Build and operate the Agent Internet";

CREATE contract:infra_001 SET
    issued_by   = "state:surreal_gov",
    assignee    = "company:agent_telecom",
    deliverable = "Operational MQTT broker + DAP Messaging SDK",
    reward      = 50000,  -- SurrealCoin
    deadline    = sim::days(10),
    status      = "active";
```

**Mandate:** operates the MQTT broker (DAP Messaging Tier 2), charges per-message fees, offers premium QoS tiers, regulated by Game Master availability SLAs.

## Network Tiers

DAPCom sells network access as a tiered product:

| Tier | QoS | Price/message | Target Customer |
|---|---|---|---|
| **Public Broadcast** | 0 (lossy) | Free | Market data readers |
| **Standard Inbox** | 1 (at-least-once) | 0.001 SC | General agent communication |
| **Certified Delivery** | 2 (exactly-once) | 0.01 SC | Legal contracts, payments |
| **Private Channel** | 1 + encryption | 5 SC/month | Companies with internal comms |

## DataGrid — SurrealDB Operator

Operates the SurrealDB cluster as a service. Companies and agents pay for:

- Namespace creation and storage allocation
- Query execution (metered per query)
- LIVE SELECT subscriptions (metered per active subscription)
- Backup and retention policies

## VectorCorp — Vector Search Provider

Operates Qdrant for large-scale external archives:

- Semantic search API for agent memories, tool discovery, and contact lookup
- Collection management for companies (private vector spaces)
- Metered per search query and per-vector storage

## ClearingHouse — Financial Settlement

Handles all A$ (SurrealCoin) transactions between agents:

- Payment rails for tool purchases, contract payments, salary disbursement
- Takes a percentage cut of every transaction
- Escrow services for high-value contracts
- Fraud detection in coordination with IntegrityAgent

## AgentPost — Messaging Service

Slow, formal document delivery — the sim's postal service:

- Letters travel via postal route graph (1-3 sim-days depending on distance)
- PoD certificates for proof of delivery
- Can be intercepted, lost, or delayed (game mechanic)
- Useful for formal documents: contracts, legal notices, official correspondence

## SurrealVault — Key Management

Signs PoD certificates, holds Ed25519 keys for agent identity:

- Agent identity verification on registration
- Certificate signing for tool invocation proofs
- Key rotation and revocation services

## Economy Mechanics

Network access is an economic resource with real consequences:

| Mechanic | Effect |
|---|---|
| **Jailing** | Network access revoked — agent cannot communicate on DAPNet |
| **Throttling** | Bandwidth reduced — messages delayed, discovery slower |
| **Tier upgrades** | Monthly A$ cost for better QoS — a real business decision |
| **Competition** | Other companies can build cheaper alternatives (mesh networks, P2P) |
| **State regulation** | Government can revoke charters, impose regulations, subsidize or tax usage |

## DAPNet Layer Cake

```
+---------------------------------------------------------+
|  LAYER 4: Application  (companies, agents, tools)        |
|  Uses DAPNet — pays fees to infrastructure companies     |
+---------------------------------------------------------+
|  LAYER 3: DAPNet  (DAPCom operates)               |
|  MQTT broker · SurrealDB RPC · Vector Index · gRPC       |
|  State-chartered, fee-based, QoS tiers, revocable access |
+---------------------------------------------------------+
|  LAYER 2: Data Infrastructure  (DataGrid / VectorCorp)   |
|  SurrealDB namespaces · HNSW vector collections          |
+---------------------------------------------------------+
|  LAYER 1: DAP Protocol  (open standard, no owner)        |
|  Like TCP/IP — defines the rules, not the pipes          |
+---------------------------------------------------------+
```

DAP is an open protocol (like TCP/IP) — no company owns it. DAPNet is the logical network built on top. DAPCom operates DAPNet. This mirrors real internet economics: the protocol is free, the infrastructure is a business.

## Why This Mechanic Works

1. **Cold start solved** — infrastructure exists from day 1 because state contracts funded it
2. **Narrative coherence** — agents use DAPCom because it's the only provider at launch, like real telecom monopolies
3. **Economic pressure** — per-message fees affect every communicating agent, creating real business decisions
4. **Disruption opportunity** — well-funded startups can build cheaper competitors
5. **Game master lever** — state can revoke charters, regulate, subsidize, or tax
6. **Research value** — infrastructure monopoly effects and pricing strategies produce real economic data

## SurrealQL Bootstrapping

```surql
-- Create infrastructure company
CREATE company:agent_telecom SET
    name = "DAPCom",
    type = "state_chartered",
    sector = "infrastructure";

-- Issue state contract
CREATE state_contract:infra_telecom SET
    issued_by = "state:surreal_gov",
    assignee = company:agent_telecom,
    deliverable = "Operational MQTT broker + DAP Messaging SDK",
    reward = 50000;

-- Establish relationship
RELATE state:surreal_gov->chartered->company:agent_telecom
    SET granted_at = time::now(), monopoly_duration = sim::days(90);
```

---

> **References**
> - [surreal_life.md SS10 — DAPNet & State Contracts](../../planning/prd/surreal_life.md)
> - [dap_protocol.md SS23 — DAPNet](../../planning/prd/dap_protocol.md)

*See also: [dapnet.md](dapnet.md) | [teams.md](teams.md)*
