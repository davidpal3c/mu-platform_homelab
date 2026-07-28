# 03 — Request path

**Target state.** Ingress, TLS, and the workload are not yet deployed. Narrative: [../architecture.md](../architecture.md).

```mermaid
sequenceDiagram
    autonumber
    participant U as User (browser / bot)
    participant I as Ingress — TLS termination
    participant A as Lemurjob API
    participant R as Redis
    participant P as PostgreSQL
    participant W as Job worker
    participant O as Observability

    U->>I: HTTPS request
    I->>I: Terminate TLS · route by host/path · rate limit
    I->>A: Forward over cluster network

    A->>A: Authenticate · authorise
    A->>P: Read user / job state
    P-->>A: Rows

    alt Long-running work
        A->>R: Enqueue job
        A-->>U: 202 Accepted — job id
        R->>W: Worker consumes from queue
        W->>P: Persist result
        W-->>O: Emit metrics + logs
    else Fast path
        A->>R: Cache lookup
        alt Cache hit
            R-->>A: Cached response
        else Cache miss
            A->>P: Query
            P-->>A: Rows
            A->>R: Populate cache with TTL
        end
        A-->>U: 200 OK
    end

    A-->>O: Emit metrics + logs
    Note over O: Request rate, error rate,<br/>latency percentiles, queue depth
```

**Design rules visible in the flow:**

- **TLS terminates at the edge.** Only the ingress controller is reachable from the internet; the Kubernetes API server never is.
- **Long work is asynchronous.** Job processing never blocks an HTTP response, so the API stays fast and predictable while workers do the expensive part.
- **Every path is observed.** Request-level signals and queue depth are emitted by default rather than added after the first incident.
- **Data services are internal.** PostgreSQL and Redis are only reachable from inside the cluster.
