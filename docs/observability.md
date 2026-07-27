# Observability

**Status: PLANNED.** Designed and sequenced; not deployed. It lands after the Kubernetes bootstrap is accepted and **before** the workload — the platform observes itself before it is trusted with something real.

---

## 1. Position in the build order

Observability is scheduled ahead of the workload on purpose. Two concrete reasons, not a slogan:

1. **The storage layer already has an unmonitored risk.** Tier mounts use `nofail`, which means a disk that fails to mount degrades silently and services start against an empty directory. The mitigation for that is a mount-presence alert. Until it exists, that risk is simply carried.
2. **Retention is a capacity problem.** Prometheus and Loki write to `/platform`. Deploying them without retention limits and disk alerts is how you fill a tier and take out the container runtime with it.

Deploying a workload first would mean debugging it blind and finding out about disk pressure from a crash.

---

## 2. Stack

| Component | Role | Storage |
|-----------|------|---------|
| Prometheus | Metrics collection, storage, alert rule evaluation | `/platform/prometheus` |
| Loki | Log aggregation | `/platform/loki` |
| Grafana | Dashboards for metrics and logs | Config as code; no critical state |
| Alertmanager | Alert routing, grouping, deduplication | Minimal |
| node-exporter | Host metrics — CPU, memory, disk, filesystem, SMART | — |
| kube-state-metrics | Kubernetes object state | — |

Deployed into the `observability` namespace. Both storage paths are already bind-mounted onto the platform tier by the storage layer — one of the reasons those binds were created before Kubernetes existed.

---

## 3. Signals that matter

### Storage — the highest-value signals on this node

| Signal | Why | Alert |
|--------|-----|-------|
| Filesystem usage per mount (`/`, `/var`, `/platform`, `/data`, `/backups`) | Every tier has a distinct fill mode and a distinct consequence | 80 % warn · 90 % high · 95 % critical |
| **Expected mountpoint present** | Directly mitigates the `nofail` silent-failure risk | Any expected tier not mounted → critical |
| SMART health and temperature | Two boot SSDs are ~90 % health; the HDD is a spinner | Any SMART failure → critical |
| RAID array state | Degraded RAID1 is survivable but must not go unnoticed | Not `[UU]` → critical |
| Prometheus / Loki retention growth rate | Observability filling the platform tier is a self-inflicted outage | Sustained growth → warn |

### Node and cluster

CPU load and steal, memory pressure and OOM kills, IO wait and per-device latency, node `Ready` status, pod restart loops, pods pending on unschedulable resources, PVC binding failures.

### Data services — once deployed

PostgreSQL: connection count against `max_connections`, replication and WAL generation rate, `/data/postgres_wal` size, transaction rate, slow queries, last successful backup age.

Redis: memory usage against `maxmemory`, evictions, queue depth per worker queue, persistence failures.

### Workload — once deployed

Request rate, error rate, and latency percentiles at the ingress; job queue depth and processing latency; job failure rate; certificate expiry.

---

## 4. Alert philosophy

An alert must be **actionable** and **have a documented response**. Anything else is noise that trains you to ignore the channel.

| Severity | Meaning | Response |
|----------|---------|----------|
| Critical | User-visible outage, or data at risk | Immediate |
| High | Degraded, or heading for critical without intervention | Same day |
| Warning | Trend needs attention | Next work session |

Initial critical set — deliberately short:

- Any expected tier mount missing
- Any filesystem above 95 %
- RAID array degraded
- SMART failure on any device
- Node `NotReady`
- PostgreSQL down, or last successful backup older than its RPO

The single-node reality is worth naming: **there is no second node to alert from.** If the node is down, the alert does not fire from the node. Off-host alerting — even something as simple as an external uptime check — is a required follow-up, not a refinement.

---

## 5. Dashboards

| Dashboard | Answers |
|-----------|---------|
| Node overview | Is the machine healthy? CPU, memory, IO, uptime |
| Storage tiers | How full is each tier, how fast is it filling, are all mounts present |
| Kubernetes cluster | Pod health, restarts, pending workloads, PVC state |
| Data services | Postgres and Redis health, connections, queue depth |
| Workload | Lemurjob request rate, errors, latency, job throughput |

Dashboards are provisioned as code and committed. A dashboard that exists only in Grafana's database disappears with the pod.

---

## 6. Acceptance

The observability phase is complete when:

1. Prometheus is scraping node, cluster, and kube-state targets with all targets `up`.
2. Grafana shows **per-tier disk usage** — the signal that motivated the phase.
3. Loki ingests container and system logs, queryable from Grafana.
4. Alertmanager routes to a real destination and a deliberately triggered test alert arrives.
5. Retention limits are configured for both Prometheus and Loki, and the resulting worst-case footprint on `/platform` is documented.
6. Mount-presence alerts are live for `/platform`, `/data`, and `/backups`.

Point 4 matters more than it looks: an alerting pipeline that has never delivered an alert is untested.

---

## 7. Deferred

- Distributed tracing (OpenTelemetry) — meaningful once there is a multi-service request path.
- SLOs and error budgets — needs a real traffic baseline first.
- Long-term metric storage beyond local retention.
- Log-based alerting on application error patterns.
