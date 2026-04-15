# Current Focus

Project: Aluna Platform Homelab  
Phase: Phase 0 — Platform Foundation  
**Sub-phase (active): Phase 0C — Kubernetes Platform Bootstrap** ([roadmap](roadmap.md))

**0A** (Hardware + OS) and **0B** (Storage architecture) are **complete**. **TASK-0001** and **TASK-0002** are **closed.** Tiered storage is live: `/platform`, `/data`, `/backups`, UUID `fstab`, bind mounts to `/var/lib/*`.

The project is now focused on **installing and stabilizing k3s** on the node, with runtime data on the platform tier paths already prepared. That work is **TASK-0003**.

Roadmap alignment:

• **0A** — done  
• **0B** — done (TASK-0002)  
• **0C** — **now** (k3s bootstrap — TASK-0003)

---

# Active Milestone

**Kubernetes platform bootstrap (Phase 0C)**

Four-tier storage is established on the node. Next capability: **functional single-node k3s** (control plane, container runtime, namespaces, networking baseline). Ingress/TLS and observability stack follow in later tasks/phases per [roadmap](roadmap.md).

Reference layout (unchanged):

BOOT TIER — TASK-0001: RAID1, md devices for `/boot` / `/` / `/var`.  
PLATFORM TIER — TASK-0002: `/platform` + bind mounts for rancher, kubelet, containerd, prometheus, loki.  
DATABASE TIER — `/data` for Postgres, WAL, Redis.  
BACKUP TIER — `/backups` on separate HDD.

---

# Current Workstream

Platform Infrastructure

Focus areas (updated):

1. ~~Hardware / Ubuntu (0A)~~ — complete  
2. ~~Storage tiers (0B, TASK-0002)~~ — complete  
3. **Kubernetes bootstrap (TASK-0003)** — immediate  
4. Observability foundation (TASK-0004) — Phase 1, after TASK-0003  
5. Ingress + TLS (TASK-0005) — after TASK-0003  

---

# Active Tasks

**Completed**

TASK-0001 — Ubuntu Server installation + RAID1 boot mirror  
TASK-0002 — Storage tier mount configuration ([tasks/TASK_0002.md](../tasks/TASK_0002.md))

**Immediate focus**

TASK-0003 — k3s cluster bootstrap — **canonical spec:** [tasks/TASK_0003.md](../tasks/TASK_0003.md); queue index: [tasks/todo.md](../tasks/todo.md); optional runbook target: `platform/k3s/cluster-setup.md`

**Later**

TASK-0004 — Observability stack baseline (Phase 1)  
TASK-0005 — Ingress controller installation  

---

# Blocked Items

None currently.

Future items may depend on:

• filesystem strategy decision (ext4 vs ZFS)
• domain name acquisition for TLS

---

# Explicitly Deferred Work

The following are not allowed during Phase 0:

• Aluna Context API implementation
• Celery ingestion pipelines
• AI enrichment services
• dataset ingestion
• API feature work

These belong to later roadmap phases.

---

# Success Condition for Current Phase

The phase completes when the platform node can:

• run k3s reliably  
• expose HTTPS ingress  
• host observability stack  
• maintain tiered storage layout  
• support deployment of a test container workload


At that point the system transitions to:

Phase 1 — Observability & Deployment Discipline