# 04 — Delivery pipeline

**Target state.** No pipeline is built yet. Narrative: [../cicd.md](../cicd.md).

```mermaid
flowchart TB
    subgraph SRC["Source"]
        APPREPO["Lemurjob repo<br/>application source"]
        PLATREPO["mu-platform repo<br/>manifests · Helm values"]
    end

    subgraph CI["CI — hosted runner"]
        LINT["lint + type check"]
        TEST["tests"]
        BUILD["build image<br/>tag = commit SHA"]
        SCAN["vulnerability scan"]
    end

    REG[("Container registry<br/>immutable · digest-pinned")]

    subgraph CD["CD — reconciliation on mu-node-01"]
        DESIRED["Read desired state from Git"]
        APPLY["Apply to cluster"]
        GATE{"Health + readiness gate"}
    end

    OBS[("Observability<br/>deploy events correlated<br/>with metrics")]

    APPREPO --> LINT --> TEST --> BUILD --> SCAN --> REG
    PLATREPO --> DESIRED
    REG --> DESIRED
    DESIRED --> APPLY --> GATE

    GATE -->|healthy| OK["Release recorded<br/>running version traceable to a commit"]
    GATE -->|unhealthy| RB["Automatic rollback<br/>to previous revision"]
    RB --> DESIRED

    APPLY -.-> NS[("k3s — namespace: lemurjob")]
    GATE -.-> OBS
    OK -.-> OBS
```

**Why it is shaped this way:**

- **Pull-based CD.** The cluster reconciles toward Git; CI never holds a `kubeconfig`. A compromised runner cannot reach the cluster.
- **Split repos.** Application source and deployment state live apart, since they have different review needs and different failure modes.
- **No `latest`.** Every image is tagged with a commit SHA, so "what's running?" has exactly one answer.
- **Rollback is a designed path**, exercised on purpose rather than improvised during an incident.
- **Deploys are observable.** Release events land next to metrics, so a regression can be traced to the change that caused it.

**Not covered by rollback:** database schema and data. Migrations stay backward-compatible for one release so an application rollback never requires a [restore](../backups.md).
