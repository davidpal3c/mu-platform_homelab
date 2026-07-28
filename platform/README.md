# platform/

Platform primitives — the infrastructure layer itself, independent of any workload.

| Directory | Contents | Status |
|-----------|----------|--------|
| `k3s/` | Cluster install configuration, bootstrap runbook, node setup | In flight |
| `networking/` | Ingress controller config, TLS/certificate issuers, DNS notes | Planned |
| `observability/` | Prometheus, Grafana, Loki, Alertmanager configuration and dashboards-as-code | Planned |
| `storage/` | Storage class definitions, tier layout reference, `fstab` records | Built — layout documented in [../docs/storage.md](../docs/storage.md) |
| `secrets/` | Secret *management* approach only. **No secret material is ever committed.** | Planned |

Directories get populated as each phase is executed. Empty ones are placeholders for work that hasn't started yet. See the roadmap in the [root README](../README.md).

Anything workload-specific belongs in [`../workloads/`](../workloads/README.md) or [`../deployments/`](../deployments/README.md) rather than here.
