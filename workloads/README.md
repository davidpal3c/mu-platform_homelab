# workloads/

Per-workload deployment state and operational runbooks.

| Workload | Description | Status |
|----------|-------------|--------|
| `lemurjob/` | Lemurjob (product name *Lemurian*) — the only workload on this platform | Planned |

Each workload directory holds deployment notes, environment expectations, operational runbooks, and references to its manifests under [`../deployments/`](../deployments/README.md).

**Application source is not here.** Lemurjob's code lives in its own repository; this platform owns how it is deployed, operated, observed, and recovered. What it requires of the platform, and in what order: [../docs/workload-lemurjob.md](../docs/workload-lemurjob.md).
