# Workload — Lemurjob

**Status: PLANNED.** Not deployed. The platform is being built to host it; the deployment happens once ingress, TLS, and observability are in place.

Lemurjob is the **only** workload this platform runs. That is a deliberate constraint, not a current limitation.

---

## 1. What it is

Lemurjob (product name **Lemurian**) is a multi-user job-application-prep service: users send a job link over Telegram or Signal, and the system produces grounded, verified application material — research, a variant-matched resume, a cover letter that has passed a verification pass — surfaced through a web control plane.

Application design, product scope, and roadmap live in the Lemurjob repository. **This document covers only what the platform owes it**, and what it demands of the platform in return.

---

## 2. Why one workload

A homelab with fifteen self-hosted services teaches you fifteen installation procedures. A homelab with one workload that people actually depend on teaches you operations.

| One real workload forces | A pile of toy services does not |
|--------------------------|--------------------------------|
| Uptime that matters to someone other than you | Restart it whenever |
| Backups with a tested restore | Reinstall and shrug |
| Alerts you cannot ignore | Muted within a week |
| Deploys without downtime | Nobody notices a five-minute gap |
| Capacity planning against real usage | Whatever fits |
| A security posture for real user data | Nothing sensitive to lose |

Lemurjob supplies the pressure that makes the platform work meaningful.

---

## 3. What it requires

Each requirement maps to a specific platform capability and phase:

| Requirement | Platform capability | Phase | Status |
|-------------|--------------------|-------|--------|
| Container scheduling | k3s single-node cluster | 0C | In flight |
| Persistent multi-tenant data | PostgreSQL on the isolated database tier | 3 | Planned |
| Queue and cache | Redis on the database tier | 3 | Planned |
| Browser-facing control plane | Ingress controller + TLS certificates | 2 | Planned |
| Async job processing | Worker deployments, queue-depth monitoring | 3 | Planned |
| Operability | Metrics, logs, alerts | 1 | Planned |
| Safe iteration | CI/CD pipeline with automatic rollback | 2 | Planned |
| Data durability | Automated backups with a verified restore | 4 | Planned |

The **inbound HTTPS** requirement is the one that shapes the platform most. An outbound-only background service could skip ingress and TLS entirely; a browser-facing control plane cannot. That single fact is why the edge phase exists.

---

## 4. Deployment shape

Target topology once deployed:

| Component | Namespace | Kind | Notes |
|-----------|-----------|------|-------|
| Control-plane API | `lemurjob` | Deployment | Behind ingress; the only externally reachable path |
| Web UI | `lemurjob` | Deployment | Static or server-rendered, behind ingress |
| Bot gateway | `lemurjob` | Deployment | Telegram / Signal integration; outbound + webhook |
| Job workers | `lemurjob` | Deployment | Horizontally scalable; consumes the Redis queue |
| PostgreSQL | `lemurjob-data` | StatefulSet | PVC pinned to `/data/postgres`; `ClusterIP` only |
| Redis | `lemurjob-data` | StatefulSet | PVC pinned to `/data/redis`; `ClusterIP` only |

Splitting the data services into `lemurjob-data` keeps the "never blanket-delete this namespace" rule structural instead of remembered. Application deployments churn; stateful services should not.

---

## 5. Platform-side responsibilities

What this repo owns:

- **Deployment artifacts** — manifests and Helm values under `deployments/`, plus workload-specific state under `workloads/`.
- **Resource policy** — requests and limits, so the workload cannot starve the control plane on a 32 GB node.
- **Storage routing** — PVCs land on the database tier, not the platform tier.
- **Network exposure** — only the ingress-fronted services are reachable; data services stay `ClusterIP`.
- **Secrets delivery** — Kubernetes secrets sourced from outside the repo; nothing committed.
- **Backup coverage** — Postgres dumps and Redis snapshots to `/backups`, with restore drills.
- **Observability coverage** — request rate, error rate, latency, queue depth, and job failure rate as first-class signals.

What this repo does **not** own: application source, schema design, product scope, or business logic. Those live with Lemurjob.

---

## 6. Sequencing

Lemurjob deployment is gated on platform readiness, in this order:

1. **k3s accepted** — cluster `Ready`, namespaces created, PVC path proven.
2. **Observability live** — metrics, logs, and alerts working, with a test alert actually delivered.
3. **Ingress + TLS** — public HTTPS terminating correctly, certificates auto-renewing.
4. **CI/CD** — a commit reaches the cluster without manual `kubectl`, and rollback has been tested.
5. **Data services** — Postgres and Redis deployed, with backups running.
6. **Workload deploy** — application components, behind ingress, under monitoring.
7. **Restore drill** — a real restore performed before the service carries data anyone cares about.

Step 7 is not optional and does not move to the end. Carrying user data without a proven restore is the failure mode this whole build order exists to avoid.

---

## 7. Definition of done

The platform has done its job when Lemurjob is:

- reachable publicly over HTTPS with valid, auto-renewing certificates,
- deployable from a commit with no manual cluster access,
- monitored — with alerts that reach a human who is not looking at a dashboard,
- backed up, with a restore that has been performed and timed at least once,
- recoverable from total node loss following a written runbook.

Not "it runs." **Operable.**
