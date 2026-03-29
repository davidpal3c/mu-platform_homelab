# Alethos Platform Roadmap

Strategic sequencing for building a production-like platform in a homelab environment.

This roadmap defines **major phases of capability development**.
It does NOT contain tasks.

Tasks are derived later and placed in `tasks/todo.md`.

---

# Core Philosophy

This project follows a **platform-first development strategy**.

Infrastructure, observability, and operational discipline must exist **before workloads**.

The homelab is intentionally designed as a **single-node private cloud** that mirrors production architecture patterns:

• failure domain isolation  
• observability-first operations  
• containerized workloads  
• async data pipelines  
• reproducible infrastructure  
• Git-backed system truth  

The platform must be able to **operate reliably before any application logic is deployed**.

---

# Ultimate Milestone

## First Public Deployment (F1)

The first deployable milestone is reached when:

Alethos API is reachable publicly via HTTPS and:

• requests return deterministic responses  
• observability is active  
• backups are verified  
• operational alerts exist  
• async ingestion works  
• platform recovery is possible  

At that point the system becomes:

**a live production-style service running from the homelab.**

---

# Phase 0 — Platform Foundation

Objective:

Build a **stable, observable, recoverable platform node**.

Nothing application-specific is allowed until the platform can:

• run containers reliably  
• expose ingress  
• store state safely  
• be monitored and recovered.

---

## Phase 0A — Hardware + OS Baseline

Goal:

Transform the workstation into a **headless platform node**.

Capabilities established:

• Ubuntu Server installation  
• disk partitioning and mount strategy  
• RAID boot resilience (if applicable)  
• SSH-only operator access  
• system updates and base packages  
• predictable filesystem layout  

Non-goals:

• Kubernetes  
• application services  
• observability stack  

---

## Phase 0B — Storage Architecture

Goal:

Implement the **mini production cluster storage model**.

Storage tiers must exist physically and logically.

Tiers:

Boot / System Tier
• OS and base services

Platform Tier
• container runtime
• observability persistence

Database Tier
• Postgres
• Redis
• WAL

Backup Tier
• backup archives
• snapshot exports

Design principles:

• isolate IO patterns  
• minimize blast radius  
• preserve recovery paths  

Storage design is defined in:

`context/Homelab_Hardware_setup.json` :contentReference[oaicite:2]{index=2}  

---

## Phase 0C — Kubernetes Platform Bootstrap

Goal:

Install and stabilize **k3s cluster foundation**.

Capabilities established:

• k3s control plane  
• container runtime  
• namespaces model  
• cluster networking  
• ingress controller  
• persistent volume strategy  

At the end of this phase the node becomes:

**a functional container platform.**

---

# Phase 1 — Observability & Operational Discipline

Objective:

The platform must **observe itself before running workloads**.

No meaningful application deployment should occur without:

metrics  
logs  
alerts.

Observability stack:

• Prometheus  
• Grafana  
• Loki  
• Alertmanager  

The stack must monitor:

• disk usage per tier  
• node health  
• container resources  
• Postgres metrics  
• Redis metrics  
• API performance  

Storage monitoring is explicitly required according to the storage context. :contentReference[oaicite:3]{index=3}  

---

## Phase 1A — Metrics Platform

Prometheus deployed and scraping:

• node exporter  
• kube metrics  
• system services  

Grafana dashboards available.

---

## Phase 1B — Logging Platform

Centralized logs:

• Loki ingestion  
• structured application logs  
• container logs

This enables incident debugging.

---

## Phase 1C — Alerting & SLOs

Alertmanager configured.

Critical alerts:

• disk usage
• node memory pressure
• Postgres health
• Prometheus ingestion failures
• platform disk saturation

Operational maturity begins here.

---

# Phase 2 — Deployment Discipline

Objective:

Establish **repeatable infrastructure deployment patterns**.

Before deploying workloads, the platform must support:

• manifests  
• Helm charts  
• namespace isolation  
• secrets strategy  
• TLS strategy  

This phase establishes the **operational contract for deployments**.

---

## Phase 2A — Networking & TLS

Capabilities:

• ingress controller routing  
• TLS certificates  
• domain mapping  
• rate limiting  

Public traffic path becomes:

Client → Ingress → API

Architecture defined in diagrams. :contentReference[oaicite:4]{index=4}  

---

## Phase 2B — Secrets & Security Model

Establish:

• secrets management  
• API key model  
• trust boundaries  
• service-to-service authentication  

Security documentation lives in:

docs/security/

---

# Phase 3 — Minimal Alethos API Workload

Objective:

Deploy the **first working service**.

The goal is **deterministic API responses**, not feature completeness.

Capabilities:

• FastAPI service container  
• health endpoint  
• Postgres connectivity  
• Redis caching  
• basic request handling  

Public endpoints:

/v1/context
/v1/datasets
/v1/health



This aligns with the architecture diagrams (docs/architecture-diagrams/)

---

# Phase 4 — Async Data Ingestion

Objective:

Introduce **data pipelines** without blocking API responses.

Architecture:

Celery workers + Redis broker.

Pipeline stages:

Fetch
Normalize
Geo index
Quality check
Cache refresh

The ingestion system populates Postgres.

API remains **read-optimized**.

---

# Phase 5 — AI Enrichment Layer

Objective:

Add **optional asynchronous AI enrichment**.

Design constraints:

AI must NEVER block the main API.

Architecture:

Workers call a self-hosted MiniCPM inference service.

Security rule:

The inference service is **private only** with no ingress. 

---

# Phase 6 — Operational Maturity

Objective:

Ensure the system behaves like a real platform.

Capabilities added:

• backup automation  
• restore verification  
• WAL archiving or replication simulation  
• snapshot testing  
• incident simulation  

Backups must live on the **backup tier**.

Recovery drills are mandatory.

---

# Phase 7 — Platform Hardening

Objective:

Increase resilience and security.

Possible improvements:

• OpenTelemetry tracing  
• cluster autoscaling experiments  
• distributed ingestion scheduling  
• object storage integration  
• multi-node expansion  

These belong to **post-MVP evolution**.

---

# Phase Alignment Summary

| Phase | Focus |
|------|------|
| 0 | Platform foundation |
| 1 | Observability discipline |
| 2 | Deployment model |
| 3 | Minimal API workload |
| 4 | Data ingestion pipelines |
| 5 | AI enrichment |
| 6 | Operational maturity |
| 7 | Platform hardening |

---

# Current Phase

Phase 0 — Platform Foundation

**Phase 0A — Hardware + OS baseline:** **complete.**  
Ubuntu Server (headless), UEFI, mdadm RAID1 boot tier, dual ESP, SSH. TASK-0001 closed.

**Phase 0B — Storage architecture:** **complete.**  
`/platform`, `/data`, `/backups`, UUID `fstab`, bind mounts for platform runtime paths. TASK-0002 closed.

**Phase 0C — Kubernetes platform bootstrap:** **active.**  
Install and stabilize k3s; ensure container runtime and kubelet data use platform-tier bind mounts (TASK-0003).

No application workloads should be implemented yet.

---

# Allowed Work Right Now

Infrastructure only (Phase 0C emphasis):

• **k3s installation and verification** — `kubectl`, node Ready, basic networking  
• **Runtime alignment** — k3s/containerd/kubelet data on existing `/platform` bind mounts per `context/project-context.json` and `tasks/TASK_0002.md`  
• **Namespaces / cluster baseline** as scoped in TASK-0003  
• **Documentation** — build report and blueprint updates when TASK-0003 completes  

Deferred until TASK-0003 is accepted: production-style ingress/TLS (TASK-0005), full observability stack install (TASK-0004 / Phase 1). Planning those remains allowed.

---

# Explicitly Deferred Work

The following must NOT begin yet:

• API development
• Celery pipelines
• AI services
• ingestion pipelines
• feature work

These belong to later phases.

---

# Strategic Goal of the Homelab

The homelab must become a **demonstrable platform engineering artifact**.
It intends to simulate real teams structure platform rollout:
infra → observability → deployment discipline → workloads

Artifacts expected:

• architecture diagrams  
• runbooks  
• infrastructure documentation  
• observability dashboards  
• backup procedures  
• deployment manifests  




