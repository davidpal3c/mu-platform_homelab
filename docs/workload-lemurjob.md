# Workload — Lemurjob

**Status: PLANNED.** Not deployed. The platform is being built to host it, and the deployment happens once ingress, TLS, and observability are in place.

Lemurjob is the **only** workload this platform runs. That's a constraint I chose, not a limitation I'm working around.

---

## 1. What it is

Lemurjob (product name **Lemurian**) is a multi-user job-application-prep service. Users send a job link over Telegram or Signal, and the system produces grounded, verified application material (research, a variant-matched resume, a cover letter that's been through a verification pass) surfaced through a web control plane.

Application design, product scope, and roadmap live in the Lemurjob repository. This document covers what the platform owes it and what it asks of the platform in return.

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

Lemurjob supplies the pressure that makes the platform work mean something.

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

The **inbound HTTPS** requirement shapes the platform more than anything else on this list. An outbound-only background service could skip ingress and TLS entirely, but a browser-facing control plane can't. That's why the edge phase exists at all.

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

Splitting the data services into `lemurjob-data` keeps the "never blanket-delete this namespace" rule structural instead of something I have to remember. Application deployments churn constantly; stateful services shouldn't.

---

## 5. Platform-side responsibilities

What this repo owns:

- **Deployment artifacts.** Manifests and Helm values under `deployments/`, plus workload-specific state under `workloads/`.
- **Resource policy.** Requests and limits, so the workload can't starve the control plane on a 32 GB node.
- **Storage routing.** PVCs land on the database tier, not the platform tier.
- **Network exposure.** Only the ingress-fronted services are reachable, and data services stay `ClusterIP`.
- **Secrets delivery.** Kubernetes secrets sourced from outside the repo, nothing committed.
- **Backup coverage.** Postgres dumps and Redis snapshots to `/backups`, with restore drills.
- **Observability coverage.** Request rate, error rate, latency, queue depth, and job failure rate treated as primary signals.

What this repo doesn't own: application source, schema design, product scope, or business logic. Those live with Lemurjob.

---

## 6. Sequencing

Lemurjob deployment is gated on platform readiness, in this order:

1. **k3s accepted.** Cluster `Ready`, namespaces created, PVC path proven.
2. **Observability live.** Metrics, logs, and alerts working, with a test alert actually delivered.
3. **Ingress + TLS.** Public HTTPS terminating correctly, certificates auto-renewing.
4. **CI/CD.** A commit reaches the cluster without manual `kubectl`, and rollback has been tested.
5. **Data services.** Postgres and Redis deployed, with backups running.
6. **Workload deploy.** Application components behind ingress, under monitoring.
7. **Restore drill.** A real restore performed before the service carries data anyone cares about.

Step 7 stays where it is and doesn't slide to the end. Carrying user data without a proven restore is the failure this whole build order exists to avoid.

---

## 7. Definition of done

The platform has done its job when Lemurjob is:

- reachable publicly over HTTPS with valid, auto-renewing certificates,
- deployable from a commit with no manual cluster access,
- monitored, with alerts that reach someone who isn't watching a dashboard,
- backed up, with a restore performed and timed at least once,
- recoverable from total node loss by following a written runbook.

The bar is operable and recoverable, not just running.
