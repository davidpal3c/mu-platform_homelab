# Diagram 3 — Request lifecycle

```mermaid
sequenceDiagram
    participant Client
    participant Access as Aluna Access
    participant API as Protected API
    participant Context as Aluna Context
    participant Mira as Aluna Mira
    participant Terra as Aluna Terra

    Client->>Access: API request with key / credentials
    Access->>Access: Validate identity, policy, limits, plan
    Access-->>Client: Reject if unauthorized / over quota
    Access->>API: Forward permitted request

    API->>Context: Query contextual / geo-aware data
    Context-->>API: Return structured context

    API-->>Client: Return application response

    API->>Mira: Emit logs / usage / operational events
    Context->>Mira: Emit contextual signals
    Mira->>Mira: Analyze patterns / detect anomalies

    Mira->>Terra: Feed interpreted data
    Context->>Terra: Feed geo/context data
    Terra-->>Client: Visual dashboards / maps / insights
```
