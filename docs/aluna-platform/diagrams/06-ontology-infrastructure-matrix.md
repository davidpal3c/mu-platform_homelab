# Diagram 6 — Ontology → infrastructure responsibility matrix

## Matrix

| Ontology layer | Product | Primary technical responsibility | Core infra components |
|----------------|---------|----------------------------------|------------------------|
| Control | Ceiba | Identity, policy, quota, monetization enforcement | API runtime, Postgres, Redis, control API |
| Meaning | Aluna Context | Normalized contextual and geo-aware data APIs | Ingestion workers, external providers, context store |
| Awareness | Aluna Mira | Telemetry interpretation and AI-assisted ops insight | Logs/metrics/events pipeline, detection engine, AI reasoning |
| Surface | Aluna Terra | Visual exploration and geo analytics UI | Dashboard frontend, presentation API, map/data rendering |

## Architectural reading

These diagrams establish a consistent model:

1. **Ceiba** is the first enforceable runtime boundary.  
2. **Aluna Context** is the structured reality layer that applications and analytics depend on.  
3. **Aluna Mira** turns operational exhaust into understanding and guidance.  
4. **Aluna Terra** is the human-facing exploration surface for context and insight.  

**Progression:** permission → meaning → awareness → visibility.
