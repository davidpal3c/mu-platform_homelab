# Diagrams

Mermaid sources for **mu-platform**. Rendered inline by GitHub, VS Code (Mermaid preview), or exported with `mmdc`.

| Diagram | Shows | Reflects |
|---------|-------|----------|
| [01-storage-tiers.md](01-storage-tiers.md) | Physical devices → filesystems → mountpoints → bind mounts | **Built** state |
| [02-cluster-topology.md](02-cluster-topology.md) | k3s namespaces, workloads, and how PVCs land on host tiers | Target state |
| [03-request-path.md](03-request-path.md) | A request from the internet to the data plane and back | Target state |
| [04-delivery-pipeline.md](04-delivery-pipeline.md) | Commit → CI → registry → cluster, with the rollback path | Target state |

Diagrams marked **Built** describe what exists on the node today. Target-state diagrams describe the designed end state and are kept in step with the phase documents in [../](../README.md) — they are not a claim that the infrastructure is running.

Narrative context: [../architecture.md](../architecture.md).
