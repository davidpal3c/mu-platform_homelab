# workloads/

Per-workload deployment state and operational runbooks.

| Workload | Description | Status |
|----------|-------------|--------|
| `lemurjob/` | Lemurjob (product name *Lemurian*) — the only workload on this platform | Planned |

Each workload directory holds deployment notes, environment expectations, operational runbooks, and references to its manifests under [`../deployments/`](../deployments/README.md).

**Application source isn't here.** Lemurjob's code lives in its own repository, and this platform owns how it gets deployed, operated, observed, and recovered. What it needs from the platform, and in what order: [../docs/workload-lemurjob.md](../docs/workload-lemurjob.md).
