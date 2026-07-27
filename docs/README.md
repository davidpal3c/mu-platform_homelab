# Documentation index

Platform documentation for **mu-platform** — the homelab platform node `mu-node-01`.

Each document states its **status** up front:

- **Built** — implemented on the node and verified.
- **In flight** — actively being executed.
- **Planned** — designed and sequenced, not yet deployed.

| Document | Scope | Status |
|----------|-------|--------|
| [architecture.md](architecture.md) | Whole-platform architecture, trust boundaries, current vs. target state | Mixed |
| [storage.md](storage.md) | Four-tier storage: boot RAID1, platform, database, backup | **Built** |
| [kubernetes.md](kubernetes.md) | Single-node k3s: runtime paths, namespaces, storage class | **In flight** |
| [observability.md](observability.md) | Prometheus, Grafana, Loki, Alertmanager; signals and alert rules | Planned |
| [backups.md](backups.md) | Backup targets, retention, restore verification, PITR direction | Planned |
| [cicd.md](cicd.md) | Image build, registry, and deployment path from commit to cluster | Planned |
| [workload-lemurjob.md](workload-lemurjob.md) | The one workload, and what it requires of the platform | Planned |
| [diagrams/](diagrams/README.md) | Mermaid sources for platform diagrams | — |
| [adr/](adr/README.md) | Architecture decision records | — |

## Conventions used across these docs

| Convention | Value |
|------------|-------|
| Node hostname | `mu-node-01` |
| Storage tiers | `/platform`, `/data`, `/backups` (plus boot RAID1) |
| Kubernetes | k3s, single node, `local-path` default storage class |
| Baseline namespaces | `lemurjob`, `lemurjob-data`, `observability` |
| Workload | Lemurjob (product name *Lemurian*) — the only workload |

## What is not here

Planning material, task specifications, build reports, review reports, and the engineering log are kept in a separate, git-ignored `_workspace/` tree and tracked in their own repository. This directory holds the architectural and operational record only.
