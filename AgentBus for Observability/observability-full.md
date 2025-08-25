# Observer Cloud for Agentic Analytics
Create and Observability fabric/mesh network - a single network for Agentic and Non-Agentic Observability.

**Components**: 
- Runtime: AI native observability runtime that resolves paths, observables, vectorization, wire-tap etc.
- Observability Plane: set up subscribers to analyze and potentially act on the observables by notification, delegation of error fix to a bot, human-in-the-loop and more
- Dynamic Dashboard using Promptable UI (open source)

## System Dataflow (With Bus)

```mermaid
flowchart LR
  %% High-level dataflow with dual planes and partitioning

  subgraph Client["User / Client"]
    UQ["User Query"]
  end

  subgraph A["Agent A"]
    A1["/run or RPC"]
  end

  subgraph BUS["Agentic Bus (transparent proxy)"]
    direction LR

    subgraph Ingress["Ingress Proxy"]
      IN["/bus/proxy/{alias}/**"]
      CAN1["Canonicalize(Request)\n+ session_id, trace_id\n+ network, company_id\n+ payload hash/excerpt"]
      TAP1["wireTap → Observer Queue (async)"]
    end

    subgraph Router["Router / Egress"]
      RES["Alias Resolution\n(agent-b, tool-llm → URL)"]
      POL["Policy/ACL (opt)\nAnomaly/PII filter (opt)"]
      EG["Egress → Target"]
      CAN2["Canonicalize(Response)\n+ latency_ms, status"]
      TAP2["wireTap → Observer Queue (async)"]
    end

    subgraph Observer["Observer Plane (read-only)"]
      Q["seda/queue"]
      SSE["SSE /monitor/sse"]
      WS["WebSocket /monitor/ws"]
      %% (prod) Kafka/NATS topics partitioned by network+company
      KAFKA["(opt) Kafka topic:\nagentbus.{network}.{company}"]
    end
  end

  subgraph B["Agent B"]
    B1["/delegate"]
  end

  subgraph TOOL["Tool / LLM"]
    T1["/generate or API"]
  end

  UQ --> A1
  A1 -- HTTP/RPC --> IN
  IN --> CAN1 --> TAP1 --> Q
  CAN1 --> RES --> POL --> EG --> B1
  B1 --> EG --> CAN2 --> TAP2 --> Q --> SSE
  Q --> WS
  TAP1 -. mirror .-> SSE
  TAP2 -. mirror .-> WS
  CAN2 --> A1

  %% Tool call leg
  A1 -- HTTP/RPC (tool) --> IN
  IN --> CAN1 --> TAP1
  CAN1 --> RES --> POL --> EG --> T1
  T1 --> EG --> CAN2 --> TAP2 --> A1

  classDef plane fill:#f8f8ff,stroke:#999,color:#333;
  classDef passive fill:#eef9ff,stroke:#88a,color:#333;
  class BUS,Ingress,Router,Observer plane;
  class Q,SSE,WS,KAFKA passive;
```

---

# Sequence — Base Case (No Bus)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant AgentA
  participant AgentB
  participant ToolLLM as Tool/LLM

  User->>AgentA: query(payload)
  AgentA->>AgentB: delegate_task(payload)
  AgentB-->>AgentA: result
  AgentA->>ToolLLM: generate(context)
  ToolLLM-->>AgentA: completion
  AgentA-->>User: final answer
```

---

# Sequence — With Bus (Transparent Proxy + Observer Mirroring)

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant AgentA
  participant Ingress as Bus Ingress
  participant Egress as Bus Egress/Router
  participant AgentB
  participant ToolLLM as Tool/LLM
  participant Observer

  User->>AgentA: query(payload)

  %% A -> B leg
  AgentA->>Ingress: POST /bus/proxy/agent-b/delegate
  Note right of Ingress: Canonicalize(Request)\nwireTap → Observer (async)
  Ingress->>Egress: forward (InOnly mirror)
  Egress->>AgentB: /delegate (bridgeEndpoint)
  AgentB-->>Egress: result
  Egress-->>Ingress: response\nCanonicalize(Response)\nwireTap → Observer
  Ingress-->>AgentA: result

  %% Observer receives both REQUEST and RESPONSE mirrors asynchronously
  Ingress--)Observer: REQUEST envelope (async)
  Egress--)Observer: RESPONSE envelope (async)

  %% A -> Tool/LLM leg
  AgentA->>Ingress: POST /bus/proxy/tool-llm/generate
  Ingress->>Egress: forward
  Egress->>ToolLLM: /generate
  ToolLLM-->>Egress: completion
  Egress-->>Ingress: response\nCanonicalize + wireTap
  Ingress-->>AgentA: completion
  Ingress--)Observer: REQUEST envelope (async)
  Egress--)Observer: RESPONSE envelope (async)

  AgentA-->>User: final answer
```

---

# Flow/Logic — Ingress/Egress Route (Java/Camel Logic)

```mermaid
flowchart TD
  %% Node Definitions with Icons, Color, and Shapes

  A([fa:fa-cloud HTTP In:<br>/bus/proxy/alias]):::api
  
  %% Subgraph 1
  subgraph AgenticBus[Agentic Bus OSS]
  B([fa:fa-scissors Strip <br> CamelHttp* headers]):::process
  C([fa:fa-magic Canonicalize - Request]):::process
  
    %% Subgraph 2
      subgraph Observability[Observer Specific]
        D{{fa:fa-robot AI Observer Pipeline: &rarr; Classifier/Resolver<br> &rarr; vectorize }}:::observer
        E([fa:fa-external-link-alt wireTap &rarr; <i>async</i>Observer's UUID Channel/API]):::observer
        F([fa:fa-eye Series of AI filters<br>+ static ACL/Security<br>Policy Enforcement]):::observer
    end
  G{{fa:fa-random Resolve and Bridge<br>&rarr; using AI Route Classifier<br>&rarr; fallback: classic bus}}:::process
  H([fa:fa-inbox Receive<br>target response]):::process
  I([fa:fa-magic Canonicalize - Response]):::process
  X([fa:fa-exclamation-triangle HTTP 502<br>Bad Gateway<br>+ mirror ERROR]):::error
end
  K([fa:fa-reply Return response<br>to caller]):::api

  %% Flow edges
  A --> B
  B --> C
  C --> D
  D -- ok --> E
  D -- fail --> X
  E --> F
  F --> G
  G -- ok --> H
  G -- fail --> X
  H --> I
  I --> E
  I --> K

  %% Styling
  classDef api     fill:#2b90d9,color:#fff,stroke:#276c9c,rounded corners,stroke-width:2px;
  classDef process fill:#f9f6a7,color:#333,stroke:#d9cd4a,rounded corners,stroke-width:2px;
  classDef observer fill:#c092f7,color:#fff,stroke:#9a70b5,rounded corners,stroke-width:2px;
  classDef ai      fill:#d1f2eb,color:#155724,stroke:#aee1d1,rounded corners,stroke-width:2px;
  classDef error   fill:#f8d7da,color:#721c24,stroke:#f5c6cb,rounded corners,stroke-width:2px;
  classDef runtime fill:#eef9ff,stroke:#88a,color:#333;
  classDef app fill:#F292F7,stroke:#88a,color:#333,rounded corners;
  class AgenticBus runtime;
  class Observability app;

```

---

# Data Model — Canonical Envelope (for Observability)

```mermaid
classDiagram
  class AgentEnvelope {
    +String event_id
    +String event_type  // REQUEST|RESPONSE|TOOL_CALL|TOOL_RESULT|ERROR|HEARTBEAT
    +String ts
    +String network     // testnet|mainnet
    +String company_id
    +String session_id
    +String trace_id
    +String span_id
    +Endpoint source
    +Endpoint target
    +Protocol protocol
    +Map~String,Object~ headers
    +String payload_excerpt
    +String hash
    +Long latency_ms
    +String status
  }
  class Endpoint {
    +String agent_id
    +String instance_id
    +String host
    +String endpoint
  }
  class Protocol {
    +String version
    +String intent
    +String cache_control
    +String signature
  }
  AgentEnvelope o-- Endpoint : source/target
  AgentEnvelope o-- Protocol
```

---

# Flow/Logic — Observer Plane Fanout

```mermaid
flowchart LR
  %% Nodes with FA icons, shapes, and color classes
  Q([fa:fa-retweet<br><b>seda:observer-mirror</b>]):::entry
  M([fa:fa-code<br>Marshal &rarr; JSON]):::process
  P{{fa:fa-code-branch<br><b>Fanout</b>}}:::branch
  SSE[/fa:fa-bolt<br><b>GET /monitor/sse/</b>/]:::sse
  WS(/fa:fa-exchange-alt<br><b>ws:// /monitor/ws</b>/):::ws
  K[/fa:fa-envelope-open <b>Kafka:</b><br>agent-network-channel#sessionID/]:::kafka

  %% Edges: label with protocols
  Q --> M --> P
  P -- "SSE" --> SSE
  P -- "WebSocket" --> WS
  P -- "Kafka" --> K

  %% Class and Styling Definitions
  classDef entry     fill:#2b90d9,color:#fff,stroke:#276c9c,rounded corners,stroke-width:2px;
  classDef process   fill:#f9f6a7,color:#333,stroke:#d9cd4a,rounded corners,stroke-width:2px;
  classDef branch    fill:#d1f2eb,color:#145a50,stroke:#25a18e,rounded corners,stroke-width:2px;
  classDef sse       fill:#ffe066,color:#614200,stroke:#ffd700,rounded corners,stroke-width:2px;
  classDef ws        fill:#cafafe,color:#023047,stroke:#219ebc,rounded corners,stroke-width:2px;
  classDef kafka     fill:#c092f7,color:#45246e,stroke:#7d5bbe,rounded corners,stroke-width:2px;```
```
---

# Dataflow — Partitioning & Multi-Network

```mermaid
flowchart TB
  subgraph Testnet["testnet"]
    TQ["Observer stream: agentbus.testnet.acme"]
    TR["Observer stream: agentbus.testnet.globex"]
  end

  subgraph Mainnet["mainnet"]
    MQ["Observer stream: agentbus.mainnet.acme"]
    MR["Observer stream: agentbus.mainnet.globex"]
  end

  IN["Ingress (any)"] -->|company_id=acme| TQ
  IN -->|company_id=globex| TR
  IN -->|network=mainnet, company=acme| MQ
  IN -->|network=mainnet, company=globex| MR
```

---

# Sequence — Error/Anomaly Path (Non-Blocking)

```mermaid
sequenceDiagram
  autonumber
  participant AgentA
  participant Ingress
  participant Egress
  participant Observer

  AgentA->>Ingress: POST /bus/proxy/agent-b/delegate
  Ingress->>Egress: forward
  Egress-->>Ingress: 502 Bad Gateway (alias missing or target down)
  Ingress--)Observer: RESPONSE envelope\nstatus=502, latency_ms, event_type=RESPONSE
  Ingress-->>AgentA: 502 (data plane unaffected by mirror)
  Note over Observer: Downstream alerting / anomaly rules trigger
```

---
### License Details

* Provider: Cascade Agentic Platform
* version: 0.0.1-beta
* License: Apache 2.0
