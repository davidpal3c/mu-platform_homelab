# Aluna Platform Ontology

**Purpose:** Canonical product and control-plane vocabulary for the Aluna platform. This file is the markdown source derived from the platform ontology; it aligns naming across docs, tasks, and diagrams.

**See also:** [diagrams/](diagrams/README.md) (platform-wide Mermaid diagrams). **Aluna Context** runtime / ingestion detail: [../aluna-context/architecture-diagrams/](../aluna-context/architecture-diagrams/architecture%20diagrams.md).

---

## 1. Core definition

**Aluna** is the logical **pre-execution decision layer**: the place where decisions are made before system actions are realized. It governs whether actions are permitted, how they are interpreted, and how they are observed and surfaced.

**System interpretation**

- Exists before application execution  
- Defines rules, constraints, and meaning  
- Acts as a control plane for API-driven systems  

---

## 2. Ontological layers

The Aluna platform is composed of **four primary domains**, each a distinct layer of system reality:

| Layer | Domain | Product | Ontological function |
|-------|--------|---------|------------------------|
| 1 | Control | **Ceiba** | Permission and constraints |
| 2 | Meaning | **Aluna Context** | Structured, contextual data |
| 3 | Awareness | **Aluna Mira** | Interprets and evaluates system behavior |
| 4 | Surface | **Aluna Terra** | Presents and visualizes system state |

---

## 3. Domain definitions

### 3.1 Ceiba

**Role:** Constraint and authorization layer (Control domain product under **Aluna**).

**Definition:** Governs who may act, under what conditions, and to what extent.

**Responsibilities:** Authentication and API keys; authorization and policy; rate limits and quotas; monetization (plans, usage-based access).

**Concept:** “Permission to exist” for a request; transition from potential action → allowed action.

### 3.2 Aluna Context

**Role:** Meaning and environmental data layer.

**Definition:** Structured, queryable representations of the environment in which actions occur.

**Responsibilities:** Geo-aware data modeling; contextual enrichment; external signal aggregation; APIs for contextual intelligence.

**Concept:** Raw inputs → interpretable reality; what the system knows about the world.

### 3.3 Aluna Mira

**Role:** Observational and interpretive layer (*mirar* — to look, to watch).

**Definition:** Evaluates system behavior by observing, analyzing, and deriving insight from actions and context.

**Responsibilities:** Observability (logs, metrics, traces); anomaly detection; behavioral analysis; AI-assisted interpretation.

**Concept:** Events → understanding; awareness of system dynamics.

### 3.4 Aluna Terra

**Role:** Representation and visualization layer.

**Definition:** Spatial and visual representations of system state for human interpretation and exploration.

**Responsibilities:** Geo-analytics dashboards; data visualization; spatial exploration; user-facing analytical interfaces.

**Concept:** Understanding → perception; how reality is seen.

---

## 4. Ontological flow

```text
Potential Action
    → [Ceiba] → Permitted Action
    → [Aluna Context] → Contextualized Action
    → [Aluna Mira] → Interpreted behavior
    → [Aluna Terra] → Visualized reality
```

---

## 5. Conceptual model

**Core principle:** Aluna defines what is allowed, what it means, how it is understood, and how it is seen.

**Layer relationships**

- Ceiba depends on policies and identity.  
- Context enriches permitted actions.  
- Mira evaluates outcomes and patterns.  
- Terra exposes insights to users.  

Each layer operates independently, composes into a unified system, and can evolve without breaking the ontology.

---

## 6. Naming system

**Aluna** is the umbrella platform. Product names include **Ceiba** (control), **Aluna Context**, **Aluna Mira**, and **Aluna Terra** — not all products use the `Aluna` + [domain] prefix; **Ceiba** names the Control layer.

| Product | Domain |
|---------|--------|
| Ceiba | Control |
| Aluna Context | Meaning |
| Aluna Mira | Awareness |
| Aluna Terra | Surface |

**Characteristics:** Short, pronounceable, conceptually grounded, non-generic; linguistically consistent (Latin / Spanish influence).

**Constraints:** Avoid mixing mythological systems; avoid overloaded jargon (“Ops”, “Data”, “Cloud”) at product level; avoid feature-level names as product names.

---

## 7. Product evolution model

| Phase | Product | Focus |
|-------|---------|--------|
| 1 | Ceiba | Entry point and monetization |
| 2 | Aluna Context | Differentiated data capabilities |
| 3 | Aluna Mira | Intelligence and observability |
| 4 | Aluna Terra | Exploration and productization |

**Expansion (examples):** Aluna Flow (traffic shaping), Aluna Plans (billing abstraction), Aluna Edge (routing / locality), Aluna Signals (internal raw ingestion, not necessarily productized).

---

## 8. Strengths and considerations

**Strengths:** Structural coherence; conceptual compression; narrative consistency; scalable addition of domains.

**Considerations:** The abstraction “Aluna” needs short functional descriptors early; keep Context scoped to contextual / geo-aware structured data; reinforce Mira with monitoring and insights messaging.

---

## 9. Final statement

**Aluna** is the system layer where decisions are made before actions become reality. **Ceiba** governs permission, **Aluna Context** defines meaning, **Aluna Mira** interprets behavior, and **Aluna Terra** reveals outcomes.

---

## 10. Mapping to this repository

| Ontology product | Homelab / infra repo role |
|------------------|---------------------------|
| Ceiba | Auth, keys, policies, quotas (workloads + edge) |
| Aluna Context | Primary API / data plane workloads (Postgres, Redis, ingestion) |
| Aluna Mira | Observability stack (Prometheus, Loki, Grafana, alerts) |
| Aluna Terra | Dashboards / geo UI (future deployment) |

**Where docs live**

- **Platform umbrella** (this ontology + platform-wide diagrams): [`docs/aluna-platform/`](README.md) (you are reading [`platform-ontology.md`](platform-ontology.md)).  
- **Aluna Context workload** (runtime architecture, request paths, ingestion, AI enrichment, k3s views — preserved from the former data-plane project): [`docs/aluna-context/architecture-diagrams/`](../aluna-context/architecture-diagrams/architecture%20diagrams.md).  

This repo (`alethos-platform` git remote) holds **platform foundation and deployment truth** for the Aluna stack on the homelab node **aluna-node-01**.
