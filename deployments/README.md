# deployments/

Cluster deployment artifacts — the declared desired state that the cluster reconciles toward.

| Directory | Contents |
|-----------|----------|
| `namespaces/` | Namespace definitions: `lemurjob`, `lemurjob-data`, `observability` |
| `manifests/` | Raw Kubernetes manifests |
| `helm/` | Helm release values for third-party charts |

## Conventions

- **Images are pinned by commit SHA or digest.** Never `latest`, for the reasons in [../docs/cicd.md](../docs/cicd.md).
- **No secret material.** Manifests reference Kubernetes secrets by name, and the values come from outside this repo.
- **Resource requests and limits are mandatory.** On a single 32 GB node, an unbounded workload can destabilise the control plane.
- **Stateful services stay in `lemurjob-data`**, with PVCs routed to the database tier.

Empty until the cluster bootstrap is accepted. Target topology: [../docs/diagrams/02-cluster-topology.md](../docs/diagrams/02-cluster-topology.md).
