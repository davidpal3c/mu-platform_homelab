# mu-platform

This is single-node homelab platform, built and operated like a small production cluster, running one real workload currently being built, [Lemurjob](docs/workload-lemurjob.md), testing uptime, backups, and a restore path start. 

This repo is the public record of the platform layer where you can find hardware and storage design, Kubernetes (k3s), observability, ingress and TLS, persistence, backups, and the path that gets a commit onto the cluster.

The homelab platform showcases a structured rollout:

> **infrastructure → observability → deployment discipline → workloads**


**Status: Phase 0, platform foundation.** 
Hardware, OS, and tiered storage are built and verified. The Kubernetes bootstrap is what's in flight. Everything past that is designed and sequenced but not deployed yet.


---

## The workload

**[Lemurjob](docs/workload-lemurjob.md)** (product name *Lemurian*) is a multi-user job-application-prep service with a web control plane and a Telegram/Signal bot layer. It's the only workload this platform is being built for right now.

Its needs are what set the platform requirements:

| Lemurjob needs | What the platform has to provide |
|----------------|----------------------------------|
| Browser-facing control plane | Real inbound HTTPS: ingress controller and TLS certificates |
| Multi-tenant user data | PostgreSQL on an isolated storage tier, with a restore path that's been tested |
| Async agent and render jobs | Redis plus worker deployments, with queue depth as a monitored signal |
| A service people actually use | Metrics, logs, and alerts |
| Iteration without downtime | A deployment pipeline instead of `kubectl apply` from a laptop |

Lemurjob's application source lives in its own repo. This one owns how it gets deployed, operated, observed, and recovered.

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

One host, but split by failure domain. OS churn, container and observability churn, database IO, and backups each get their own device, so a full disk or a bad deploy stays contained. Details in [docs/storage.md](docs/storage.md).

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

Every doc says up front whether it's `built`, `in flight`, or `planned`. Nothing here describes infrastructure that doesn't exist yet.

---

## Repo layout

```
docs/           architecture, platform docs, diagrams, ADRs
platform/       platform primitives — k3s, networking, storage, observability, secrets
deployments/    cluster deployment artifacts — manifests, helm values, namespaces
workloads/      per-workload deployment state and runbooks (Lemurjob)
```

`_workspace/` is git-ignored. Planning docs, task specs, build reports, and the engineering log live there and are tracked separately, so this repo stays the operational and architectural record rather than the scratchpad.

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

The milestone I'm working toward: Lemurjob reachable publicly over HTTPS, observability running, alerts wired somewhere I'll actually see them, and a restore I've done at least once. Getting it running is the easy half.

---

## Operating principle

**The repo is the memory.**

Architecture, deployment details, and the reasoning behind decisions get written down here instead of living in my head or in a chat log. Anyone picking this up, me included six months from now, should be able to reload the state of the platform from what's committed.
