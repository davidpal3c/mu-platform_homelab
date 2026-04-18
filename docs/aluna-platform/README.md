# Aluna platform documentation (umbrella)

This directory holds **platform-wide** material: the **Aluna** ontology (pre-execution decision layer) and **cross-product** Mermaid diagrams. It does **not** replace workload-specific docs.

| Resource | Path |
|----------|------|
| **Platform ontology** (Ceiba, Aluna Context, Mira, Terra — naming and layers) | [platform-ontology.md](platform-ontology.md) |
| **Platform-level Mermaid diagrams** (layering, lifecycle, infra mapping) | [diagrams/](diagrams/README.md) |
| **Original text sources** (operator snapshots) | [source/](source/) |

## Workload-specific documentation

Future and current products live in **separate folders** under `docs/`:

| Workload | Purpose | Documentation |
|----------|---------|---------------|
| **Aluna Context** | Meaning layer — geo-aware / contextual API, ingestion, data plane | [../aluna-context/README.md](../aluna-context/README.md) |
| Ceiba | Control layer | *TBD — manifests / runbooks in repo when added* |
| Aluna Mira | Awareness layer | *TBD* |
| Aluna Terra | Surface layer | *TBD* |

**Aluna Context** retains the existing **runtime architecture** narrative (FastAPI paths, Celery, k3s, PNG exports): [../aluna-context/architecture-diagrams/](../aluna-context/architecture-diagrams/architecture%20diagrams.md).

## Naming

- **Aluna** — umbrella platform and ontology.  
- **Ceiba / Aluna Context / Aluna Mira / Aluna Terra** — products (this folder describes all four; **Context** has its own subtree above).  
- **aluna-node-01** — canonical homelab hostname in infra docs.  
- **Git remote** — may still be `alethos_platform`; content here describes the **Aluna** platform.

## Pointers

- Stateful workflow: [context/AGENTS.md](../../context/AGENTS.md)  
- Live homelab storage: [context/Final_disk_layout_all_drives.md](../../context/Final_disk_layout_all_drives.md), [system-blueprint.md](../system-blueprint.md)  
