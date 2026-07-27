# Kubernetes — k3s

**Status: IN FLIGHT.** The cluster bootstrap is the task currently being executed. Storage prerequisites are complete; nothing below is claimed as running until the bootstrap is verified on the node.

---

## 1. Why k3s

Single node, 32 GB of RAM, and a workload that needs a real cluster rather than a simulation of one:

| Requirement | Why k3s fits |
|-------------|--------------|
| Low control-plane overhead | Single binary, embedded SQLite datastore, modest memory footprint |
| Genuine Kubernetes API | Conformant — manifests, Helm charts, and operator patterns transfer to any cluster |
| Batteries included, removable | Ships Traefik, ServiceLB, `local-path`, CoreDNS; each can be disabled and replaced |
| Sane storage story | `local-path` provisioner maps PVCs onto the host's own tiers |

Full kubeadm would spend a meaningful share of this node's memory on control-plane components that buy nothing on one machine. Docker Compose would skip exactly the parts worth practising: scheduling, PVC lifecycle, ingress, namespace isolation, and rollout mechanics.

---

## 2. Runtime paths

The k3s runtime writes to paths already bind-mounted onto the platform tier by the storage layer:

| k3s path | Backed by |
|----------|-----------|
| `/var/lib/rancher` | `/platform/rancher` |
| `/var/lib/kubelet` | `/platform/kubelet` |
| `/var/lib/containerd` | `/platform/containerd` |

This is why storage came first. Container image layers and kubelet state are among the highest-churn data on the node, and none of it belongs on a RAID1 boot mirror built from decade-old SATA SSDs.

Verified — not assumed — with:

```bash
findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd
```

---

## 3. Namespace baseline

| Namespace | Contents |
|-----------|----------|
| `lemurjob` | Application workloads — control-plane API, web UI, bot gateway, job workers |
| `lemurjob-data` | Stateful services — PostgreSQL, Redis |
| `observability` | Prometheus, Grafana, Loki, Alertmanager |

Deliberately small. Three namespaces map to three genuinely different operational profiles: stateless app workloads that get redeployed constantly, stateful services that must not be casually restarted, and the monitoring stack that has to keep working while the other two are broken.

Splitting data into its own namespace is the load-bearing decision — it makes "do not blanket-delete this namespace" a structural property instead of a thing you have to remember at 2 a.m.

System components (CoreDNS, Traefik, `local-path-provisioner`, metrics-server) stay in `kube-system` as k3s ships them.

---

## 4. Storage classes

`local-path` is the default provisioner and is appropriate here: single node, no distributed storage, PVs bound to a directory on the host.

| Concern | Position |
|---------|----------|
| Provisioner | k3s `local-path` (default) |
| PV backing path | Platform tier by default; stateful services pinned to database-tier paths |
| Node affinity | Irrelevant on one node — but a real constraint if this ever grows |
| Resize | Not supported by `local-path`; capacity is planned up front |
| Backup | PVC contents are backed up at the application layer (`pg_dump`, Redis snapshots), never by copying live PV directories |

Postgres and Redis PVCs resolve to the database tier so that their IO stays isolated from container and observability churn. That routing is the whole reason `/data` exists as a separate device.

---

## 5. Bootstrap acceptance

The bootstrap is not accepted until all of the following hold on the live node:

1. `kubectl get nodes` reports the node `Ready`.
2. `kubectl get pods -A` shows core system pods healthy.
3. Baseline namespaces exist exactly as `lemurjob`, `lemurjob-data`, `observability`.
4. `findmnt` confirms `/var/lib/rancher`, `/var/lib/kubelet`, and `/var/lib/containerd` are backed by `/platform/*`.
5. A smoke-test PVC binds and a test pod mounts and writes to it.
6. `cat /proc/mdstat` confirms the boot RAID is unchanged — `[UU]`.

Rollback is the k3s uninstall script, followed by re-verifying that tier mounts and RAID state are exactly as they were before.

---

## 6. Constraints held during bootstrap

| Constraint | Reason |
|------------|--------|
| No `mkfs`, no repartitioning | Storage tiers are verified state; the cluster bootstrap has no business touching block devices |
| Boot RAID untouched (`sda`, `sdc`, `md*`) | Out of scope, full stop |
| No public ingress, no exposed API server | Edge exposure is a separate, later phase with its own hardening review |
| No workload deployment | The cluster proves itself with a PVC smoke test, not with an application |

---

## 7. Deferred to later phases

- **Ingress controller and TLS** — Traefik ships with k3s but stays internal until the edge phase; the ingress choice, ACME configuration, and exposure review happen there.
- **Observability stack** — deployed after the cluster is accepted, and before any workload. See [observability.md](observability.md).
- **Network policy** — default-allow today. Namespace-level policy is a hardening-phase deliverable.
- **Resource quotas and limits** — needed once real workloads compete with the control plane for 32 GB.
- **Backup of cluster state** — k3s etcd/SQLite snapshots to the backup tier. See [backups.md](backups.md).

---

## 8. Single-node realities

Stated plainly rather than papered over:

- **No high availability.** One control plane. Node loss is downtime, and the recovery story is restore-from-backup — which is why restore rehearsal is a first-class deliverable rather than an aspiration.
- **Control plane and workloads share resources.** A memory-hungry workload can destabilise the API server. Limits and quotas are how that gets bounded.
- **`local-path` PVs are node-local.** Fine on one node; a migration problem the day a second node exists. Documented now so it is not a surprise later.
