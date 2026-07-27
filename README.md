# mu-platform

A single-node **homelab platform** built and operated like a small production cluster — and used to run one real, live workload rather than a lab full of toys.

This repo is the public, operational record of the platform layer: hardware and storage design, Kubernetes (k3s), observability, ingress and TLS, persistence, backups, and the CI/CD path that gets a workload from commit to cluster.

**Status: Phase 0 — platform foundation.** Hardware, OS, and tiered storage are built and verified. Kubernetes bootstrap is the task in flight. Everything past that is designed and sequenced but not yet deployed — this README says so explicitly wherever it applies, and so do the docs.

---

## Why this exists

Most homelab repos are a pile of `docker-compose` files. This one is deliberately structured the way a platform team would sequence a real rollout:

> **infrastructure → observability → deployment discipline → workloads**

The platform has to be able to run, observe, and recover itself *before* an application is allowed on it. That ordering is the point of the project, and it is enforced by the task queue: application work stays blocked until the platform capability it depends on is accepted.

The second design rule is that there is **one workload, and it is real**. Platform engineering only teaches you something when something actually depends on it — uptime, backups, and a restore path start to matter when a live service is on the other side.

---

## The workload

**[Lemurjob](docs/workload-lemurjob.md)** (product name *Lemurian*) — a multi-user job-application-prep service with a web control plane and a Telegram/Signal bot layer. It is the sole workload this platform is being built for.

Lemurjob is the *reason* for the platform requirements, not an afterthought:

| Lemurjob needs | Platform capability it forces |
|----------------|-------------------------------|
| Browser-facing control plane | Real inbound HTTPS — ingress controller + TLS certificates |
| Multi-tenant user data | PostgreSQL on an isolated storage tier, with a verified restore path |
| Async agent / render jobs | Redis + worker deployments, queue depth as a monitored signal |
| A service people actually use | Metrics, logs, alerts — you cannot operate blind |
| Iteration without downtime | A deployment pipeline, not `kubectl apply` from a laptop |

Application source for Lemurjob lives in its own repo. **This repo owns how it is deployed, operated, observed, and recovered** — not what it does.

---

## Hardware

| Component | Detail |
|-----------|--------|
| Node | Lenovo ThinkStation P510 — `mu-node-01` |
| CPU / RAM | Xeon E5-2640 v4 · 32 GB |
| OS | Ubuntu Server LTS, headless, SSH-only |
| Kubernetes | k3s, single node |
| Boot | 2 × ~180 GB Intel SATA SSD, mdadm RAID1, dual ESP |
| Platform tier | ~238 GB NVMe → `/platform` |
| Database tier | ~477 GB NVMe → `/data` |
| Backup tier | ~931 GB SATA HDD → `/backups` |

One host, but partitioned by **failure domain**: OS churn, container/observability churn, database IO, and backups each get their own device, so a full disk or a bad deploy has a bounded blast radius. Details: [docs/storage.md](docs/storage.md).

---

## Documentation

| Document | What it covers |
|----------|----------------|
| [docs/architecture.md](docs/architecture.md) | As-built platform architecture, trust boundaries, diagrams |
| [docs/storage.md](docs/storage.md) | Four-tier storage design, mounts, failure domains — **built** |
| [docs/kubernetes.md](docs/kubernetes.md) | k3s topology, namespaces, storage classes — **in flight** |
| [docs/observability.md](docs/observability.md) | Prometheus / Grafana / Loki / Alertmanager design — **planned** |
| [docs/backups.md](docs/backups.md) | Backup, retention, and restore-verification model — **planned** |
| [docs/cicd.md](docs/cicd.md) | Build, registry, and deployment pipeline — **planned** |
| [docs/workload-lemurjob.md](docs/workload-lemurjob.md) | The workload and what it requires of the platform |
| [docs/diagrams/](docs/diagrams/README.md) | Mermaid sources for the diagrams above |
| [docs/adr/](docs/adr/README.md) | Architecture decision records |

Every doc carries an explicit **status** — `built`, `in flight`, or `planned`. Nothing here describes infrastructure that does not exist.

---

## Repo layout

```
docs/           architecture, platform docs, diagrams, ADRs
platform/       platform primitives — k3s, networking, storage, observability, secrets
deployments/    cluster deployment artifacts — manifests, helm values, namespaces
workloads/      per-workload deployment state and runbooks (Lemurjob)
```

`_workspace/` is git-ignored. Planning docs, task specs, build reports, and engineering logs live there and are tracked separately — this repo stays the operational and architectural record, not the scratchpad.

---

## Roadmap

| Phase | Capability | Status |
|-------|-----------|--------|
| 0A | Hardware + OS baseline — Ubuntu Server, RAID1 boot, SSH | **Complete** |
| 0B | Storage architecture — four tiers, UUID `fstab`, bind mounts | **Complete** |
| 0C | Kubernetes bootstrap — single-node k3s, namespaces, PVC path | **In flight** |
| 1 | Observability — metrics, logs, alerts before workloads | Planned |
| 2 | Deployment discipline — ingress, TLS, secrets, CI/CD | Planned |
| 3 | Lemurjob deployment — first live service | Planned |
| 4 | Operational maturity — backup automation, restore drills, PITR | Planned |
| 5 | Hardening — tracing, network policy, capacity work | Planned |

The milestone that matters: **Lemurjob reachable publicly over HTTPS, with observability active, alerts wired, and a restore that has actually been tested.** Not "it runs on my machine" — a service that can be operated and recovered.

---

## Operating principle

**The repo is the memory.**

Architecture, deployment truth, and decisions are written to files rather than living in someone's head or a chat log. Anyone — or any agent — should be able to reload the full state of this platform from what is committed here.
