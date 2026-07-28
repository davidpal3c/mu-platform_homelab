# 02 — Cluster topology

**Target state.** The cluster bootstrap is in flight; workloads are not deployed. Narrative: [../kubernetes.md](../kubernetes.md).

```mermaid
flowchart TB
    subgraph K3S["k3s — single node · mu-node-01"]
        subgraph SYS["kube-system"]
            COREDNS["CoreDNS"]
            TRAEFIK["Traefik — ingress"]
            LP["local-path-provisioner"]
            MS["metrics-server"]
        end

        subgraph APP["Namespace: lemurjob"]
            API["control-plane API"]
            WEB["web UI"]
            BOT["bot gateway"]
            WORK["job workers"]
        end

        subgraph DATA["Namespace: lemurjob-data"]
            PG["postgres — StatefulSet"]
            RD["redis — StatefulSet"]
        end

        subgraph OBS["Namespace: observability"]
            PROM["prometheus"]
            LOKI["loki"]
            GRAF["grafana"]
            AM["alertmanager"]
            NE["node-exporter"]
            KSM["kube-state-metrics"]
        end
    end

    subgraph HOST["Host storage tiers — built"]
        PLAT[("/platform")]
        DBT[("/data")]
        BKP[("/backups")]
    end

    API --> PG
    API --> RD
    WORK --> RD
    WORK --> PG
    BOT --> RD

    PG -->|PVC| DBT
    RD -->|PVC| DBT
    PROM -->|PVC| PLAT
    LOKI -->|PVC| PLAT

    TRAEFIK --> WEB
    TRAEFIK --> API
    TRAEFIK --> BOT

    NE -.-> PROM
    KSM -.-> PROM
    API -.-> PROM
    WORK -.-> PROM
    PG -.-> PROM
    RD -.-> PROM
    PROM --> GRAF
    PROM --> AM
    LOKI --> GRAF

    PG -.->|scheduled dumps| BKP
    RD -.->|snapshots| BKP
    K3S -.->|cluster snapshots| BKP
```

**Three namespaces, three operational profiles:** application workloads that redeploy constantly, stateful services that shouldn't be casually restarted, and a monitoring stack that has to keep working while the other two are broken.

**Storage routing is the detail doing the most work here.** Postgres and Redis PVCs resolve to the database tier, Prometheus and Loki to the platform tier. Neither competes with the other for IO, which is why those are separate devices.

**Data services are `ClusterIP` only.** Nothing in `lemurjob-data` is reachable from outside the cluster.
