# AEGISX Architecture Diagrams

## 1. System Architecture

```mermaid
flowchart TD
    A[Endpoint: AEGISX Agent] -->|HTTPS + JWT| B[Ingestion API - FastAPI]
    B --> C[Validation and Normalization]
    C --> D[(Redis Streams)]
    D --> E[Event Processor: Rules]
    D --> F[Event Processor: Statistics and ML]
    D --> G[Threat Intel Lookup]
    E --> H[Correlation Service]
    F --> H
    G --> H
    H --> I[Risk Engine]
    I --> J[Incident Engine]
    J --> K[Response Orchestrator]
    K --> L[Isolate]
    K --> M[Block]
    K --> N[Remediate]
    L --> O[Recovery]
    M --> O
    N --> O
    O --> P[SOC Dashboard]
    J --> P
    I --> P
```

## 2. Security Layers

```mermaid
flowchart TB
    L8[Layer 8: Cloud / External Infrastructure] --> L7[Layer 7: Identity / Access]
    L7 --> L6[Layer 6: Applications / APIs]
    L6 --> L5[Layer 5: Servers / Databases]
    L5 --> L4[Layer 4: Network]
    L4 --> L3[Layer 3: Employee Endpoints]
    L3 --> L2[Layer 2: Operating System]
    L2 --> L1[Layer 1: Kernel / Low-Level Telemetry]
```

## 3. Agent Architecture

```mermaid
flowchart LR
    Kernel[Kernel: syscalls] --> EBPF[eBPF Programs]
    Kernel --> Audit[auditd]
    EBPF --> Collector[Agent Collector]
    Audit --> Collector
    ProcFS[procfs / inotify fallback] --> Collector
    Collector --> Normalizer[Local Normalizer]
    Normalizer --> Buffer[(Local Buffer / SQLite)]
    Buffer --> Shipper[Batch Shipper]
    Shipper -->|HTTPS + JWT, retry/backoff| Ingestion[AEGISX Ingestion API]
```

## 4. Telemetry Flow

```mermaid
sequenceDiagram
    participant Agent
    participant API as Ingestion API
    participant Redis as Redis Streams
    participant Worker as Detection Worker
    Agent->>API: POST /api/v1/telemetry (batch, JWT)
    API->>API: validate schema + derive tenant from token
    API->>Redis: XADD tenant:{id}:telemetry
    API-->>Agent: 202 Accepted
    Redis-->>Worker: XREADGROUP
    Worker->>Worker: process (idempotent by event_id)
    Worker->>Redis: XACK
```

## 5. Detection Flow

```mermaid
flowchart TD
    E[Normalized Event] --> R[Rule Engine]
    E --> S[Statistical Baseline]
    S --> ML[Isolation Forest]
    E --> TI[Threat Intel Match]
    R --> F[Findings]
    ML --> F
    TI --> F
    F --> C[Correlation: group by asset/time/actor]
    C --> RE[Risk Engine]
```

## 6. AI Flow

```mermaid
flowchart LR
    T[Telemetry] --> FE[Feature Extraction]
    FE --> BL[Rolling Baseline per asset]
    BL --> AD[Isolation Forest Anomaly Score]
    AD --> BA[Behavior Analysis / Correlation]
    BA --> CO[Confidence Estimation]
    CO --> RS[Risk Signal: score + top contributing features]
```

## 7. Incident Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Detected
    Detected --> Correlated
    Correlated --> Classified
    Classified --> RiskAssessed
    RiskAssessed --> Created
    Created --> Investigating
    Investigating --> Contained
    Contained --> Remediating
    Remediating --> Verifying
    Verifying --> Recovered
    Recovered --> Reported
    Reported --> Closed
    Investigating --> Closed: False Positive
    Closed --> [*]
```

## 8. Response Lifecycle

```mermaid
flowchart TD
    D[Detection] --> RI[Risk Score]
    RI --> P{Policy Evaluation}
    P -->|Low| Mon[Monitor only]
    P -->|Medium| Alert[Alert analyst]
    P -->|High| Appr{Human Approval Required}
    P -->|Critical + policy allows| Auto[Controlled Autonomous Containment]
    Appr -->|Approved| Act[Execute Response Action]
    Auto --> Act
    Act --> Ver[Verify Action]
    Ver --> Rec[Recovery]
```

## 9. MSSP Multi-Tenancy

```mermaid
flowchart TD
    AEGISX[AEGISX Platform] --> A[Client A]
    AEGISX --> B[Client B]
    AEGISX --> C[Client C]
    A --> A1[Assets/Events/Incidents/Policies - RLS scoped]
    B --> B1[Assets/Events/Incidents/Policies - RLS scoped]
    C --> C1[Assets/Events/Incidents/Policies - RLS scoped]
```

## 10. Database Relationships (Core Entities)

```mermaid
erDiagram
    ORGANIZATIONS ||--o{ USERS : has
    ORGANIZATIONS ||--o{ ASSETS : owns
    ORGANIZATIONS ||--o{ SECURITY_ZONES : defines
    ORGANIZATIONS ||--o{ SECURITY_POLICIES : configures
    ASSETS ||--o{ AGENTS : runs
    ASSETS }o--|| SECURITY_ZONES : "located in"
    AGENTS ||--o{ TELEMETRY_EVENTS : emits
    TELEMETRY_EVENTS ||--o{ PROCESS_EVENTS : specializes
    TELEMETRY_EVENTS ||--o{ NETWORK_EVENTS : specializes
    TELEMETRY_EVENTS ||--o{ AUTHENTICATION_EVENTS : specializes
    TELEMETRY_EVENTS ||--o{ FILE_EVENTS : specializes
    TELEMETRY_EVENTS ||--o{ AI_PREDICTIONS : scored_by
    ASSETS ||--o{ RISK_SCORES : has
    RISK_SCORES ||--o{ INCIDENTS : triggers
    INCIDENTS ||--o{ INCIDENT_EVENTS : contains
    INCIDENTS ||--o{ RESPONSE_ACTIONS : triggers
    RESPONSE_ACTIONS ||--o{ REMEDIATION_ACTIONS : leads_to
    INCIDENTS ||--o{ THREAT_INDICATORS : references
    USERS ||--o{ ROLES : assigned
    ROLES ||--o{ PERMISSIONS : grants
    ORGANIZATIONS ||--o{ AUDIT_LOGS : records
```

## 11. Deployment Architecture

```mermaid
flowchart TD
    subgraph Docker Compose
        FE[Next.js Frontend]
        BE[FastAPI Backend]
        WK[Detection/AI Workers]
        PG[(PostgreSQL)]
        RD[(Redis)]
        NG[Nginx Reverse Proxy]
    end
    Client((Browser)) --> NG
    Agent((Linux Agent)) --> NG
    NG --> FE
    NG --> BE
    BE --> PG
    BE --> RD
    WK --> RD
    WK --> PG
```

## 12. Trust Boundaries

```mermaid
flowchart TD
    subgraph Untrusted
        Endpoint[Client Endpoint / Agent Host]
    end
    subgraph SemiTrusted [Semi-Trusted - Authenticated Agent]
        AgentProc[Agent Process]
    end
    subgraph Trusted [Trusted Platform - Internal Network]
        Ingestion[Ingestion API]
        Workers[Detection/Risk/Incident Workers]
        DB[(PostgreSQL)]
        Cache[(Redis)]
        Orchestrator[Response Orchestrator]
    end
    subgraph HighPrivilege [High-Privilege Boundary]
        ResponseExec[Response Executors]
    end
    Endpoint --> AgentProc
    AgentProc -->|mTLS/JWT, validated| Ingestion
    Ingestion --> Workers
    Workers --> DB
    Workers --> Cache
    Workers --> Orchestrator
    Orchestrator -->|policy + approval gate| ResponseExec
    ResponseExec -->|contain/isolate| Endpoint
```
