# Aluna Context — workload documentation

This folder is **only** for the **Aluna Context** product (meaning / geo-aware contextual data layer): runtime architecture, request flows, ingestion, and deployment-oriented diagrams that predate the full **Aluna** umbrella naming.

It is **not** the home of platform-wide ontology or cross-product diagrams — those live under **[`docs/aluna-platform/`](../aluna-platform/README.md)**.

## Contents

| Path | Description |
|------|-------------|
| [architecture-diagrams/](architecture-diagrams/architecture%20diagrams.md) | Mermaid source + numbered sections; companion **PNG** exports alongside (high-level runtime, request path, Celery ingestion, AI enrichment, k3s namespaces) |

## Relationship to other Aluna products

| Product | Doc location (when present) |
|---------|----------------------------|
| **Aluna** (umbrella ontology + platform diagrams) | [`docs/aluna-platform/`](../aluna-platform/README.md) |
| **Aluna Context** (this workload) | `docs/aluna-context/` ← you are here |
| Ceiba / Aluna Mira / Aluna Terra | Future: add `docs/ceiba/`, `docs/aluna-mira/`, `docs/aluna-terra/` or equivalent as workloads mature |

## Infra note

Homelab storage and k3s layout for **all** workloads are still described in [system-blueprint.md](../system-blueprint.md) and [context/Final_disk_layout_all_drives.md](../../context/Final_disk_layout_all_drives.md). The `/data` tier hosts **Aluna Context** state (Postgres, Redis, ingestion) per platform design.
