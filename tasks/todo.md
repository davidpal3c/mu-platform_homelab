# Task Queue

Execution-ready tasks derived from the roadmap.

---

# TASK-0001

Title: Ubuntu Server installation + RAID1 boot mirror

Phase: Phase 0 — Platform Foundation

Status: Complete

Dependencies:
None

Goal:

Install Ubuntu Server on the boot tier and implement the UEFI + mdadm RAID1 boot mirror partition plan.

Scope:

• Install Ubuntu Server (headless)
• Configure mdadm RAID1 for boot drives in UEFI mode
• Create identical GPT partitions on both SSDs for:

  /boot/efi  
  /boot  
  /  
  /var

• Create mdadm RAID1 arrays:

  /dev/md0 → /boot  
  /dev/md1 → /  
  /dev/md2 → /var

• Install GRUB on both disks
• Verify boot resilience

Non-scope:

• Kubernetes
• observability
• workload containers

Files / Docs affected:

context/project-context.json  
docs/platform-overview.md  
docs/system-blueprint.md  
docs/build-reports/TASK-0001.md

Acceptance Criteria:

• System boots from RAID1 mirror
• mountpoints match design
• SSH access working
• OS disk usage monitored

Verification Steps:

• lsblk
• cat /proc/mdstat
• df -h
• reboot test

---

# TASK-0002

Title: Storage tier mount configuration (Phase 0B)

Phase: Phase 0 — Platform Foundation (sub-phase **0B — Storage Architecture**)

Status: **Complete**

Why now:

0A is complete; k3s (0C) must not run container/runtime data on the boot RAID. Platform, database, and backup tiers must exist with UUID `fstab` and bind mounts before TASK-0003.

Dependencies:

TASK-0001 (complete)

Goal:

Implement physical/logical storage for platform, database, and backup tiers on **aluna-node-01**, aligned with `context/project-context.json`.

Scope:

• **Discover** block devices with `lsblk -f` / `nvme list` (names can vary); confirm before destructive steps.  
• **Platform tier:** single partition on 240GB WD SN530 NVMe → ext4 → `/platform`; create subdirs; bind-mount to `/var/lib/rancher`, `kubelet`, `containerd`, `prometheus`, `loki`.  
• **Database tier:** ext4 on **larger NVMe** (`/dev/nvme0n1` on aluna-node-01, ~477G 512G-class—not literal 1TB) → `/data`; create `/data/postgres`, `postgres_wal`, `redis`.  
• **Backup tier:** partition 1TB HDD → ext4 → `/backups`.  
  **Live mapping (TASK-0001):** RAID boot = **sda + sdc**; **backup HDD = sdb** — do **not** use sdc for backups (sdc is RAID member).

Non-scope:

• k3s install, ZFS/snapshot policy, backup automation, observability install.

Risks:

• Wrong disk selection can destroy RAID or OS — verify UUIDs and model/size before `mkfs`.  
• Bind mounts before k3s: empty dirs OK; ensure `/etc/fstab` ordering (base mount before bind).

Files / docs affected:

`context/project-context.json` (post-verify)  
`docs/system-blueprint.md`  
`docs/platform-overview.md`  
`docs/build-reports/BR-TASK-0002.md`

Acceptance Criteria:

• `/platform`, `/data`, `/backups` mounted at boot via UUID in `/etc/fstab`.  
• All five bind targets present and mounted after reboot.  
• `df -h` shows expected capacities; boot RAID untouched.

Verification:

`lsblk -f` · `findmnt` · `mount | grep -E 'platform|/data|backups'` · reboot test · optional `sudo systemd-analyze blame` if boot delay

Builder output:

Operator runbook (partition, mkfs, fstab, bind mounts, verification); no execution on node by Builder unless operator delegates.

---

# TASK-0003

Title: k3s cluster bootstrap

Phase: Phase 0 — Platform Foundation (**sub-phase 0C — Kubernetes Platform Bootstrap**)

Status: Ready

Dependencies:

TASK-0002 (complete)

Why now:

Phase 0B storage is complete and verified. k3s can now be installed with runtime and high-churn paths already moved to `/platform` via bind mounts, preserving boot-tier isolation.

Goal:

Install and stabilize single-node k3s on aluna-node-01, aligned with storage and failure-domain design.

Scope:

• install k3s control plane on Ubuntu Server node  
• verify runtime paths resolve to `/platform`-backed bind mounts (`/var/lib/rancher`, `kubelet`, `containerd`)  
• establish baseline namespaces: `aluna-access`, `aluna-context`, `aluna-mira`, `aluna-terra`, `observability`  
• validate cluster networking and DNS baseline  
• define initial local-path PV posture for single-node use (no production HA claims)

Naming decision (locked for TASK-0003 execution):

Use `aluna-*` namespaces only. Do not create new `opencontext-*` namespaces.

Non-scope:

TLS  
observability stack
workload deployment

Risks:

• k3s install may silently use wrong paths if bind mounts are missing or not mounted at boot.  
• resource pressure on single node if defaults are not checked (disk and memory).

Files affected:

platform/k3s/cluster-setup.md  
docs/system-blueprint.md  
docs/build-reports/BR-TASK-0003.md
context/project-context.json (status + cluster baseline)

Acceptance Criteria:

• `kubectl get nodes` shows single node `Ready`  
• `kubectl get pods -A` shows healthy system pods (CoreDNS, metrics components if installed by default, local-path provisioner)  
• namespace set created and listed  
• `/var/lib/rancher`, `/var/lib/kubelet`, `/var/lib/containerd` confirmed mapped to `/platform/*` via `findmnt`  
• basic test PVC can bind and mount using local-path storage class

Verification:

`kubectl get nodes -o wide`  
`kubectl get pods -A -o wide`  
`kubectl get ns`  
`findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd`  
`kubectl get sc`  
PVC smoke test apply/get/describe

Builder output:

Operator runbook for k3s install, post-install checks, namespace bootstrap, PVC smoke test, and rollback notes.

---

# TASK-0004

Title: Observability stack baseline

Phase: Phase 1 — Observability Discipline

Status: Pending

Dependencies:

TASK-0003

Goal:

Deploy Prometheus + Grafana + Loki stack.

Scope:

• Prometheus deployment
• Grafana dashboards
• Loki logging
• node-exporter metrics

Acceptance Criteria:

• metrics visible in Grafana
• disk usage metrics visible
• logs ingested

Verification:

Grafana dashboards functional.

---

# TASK-0005

Title: Ingress controller + TLS

Phase: Phase 2 — Deployment Discipline

Status: Pending

Dependencies:

TASK-0003

Goal:

Establish public traffic routing.

Scope:

• ingress controller installation
• TLS certificate management
• domain routing

Acceptance Criteria:

• HTTPS endpoint accessible
• routing to internal services works

Verification:

curl https://domain/health

---

# TASK-0006

Title: Runtime naming convergence (post-bootstrap, non-destructive)

Phase: Phase 1 — Observability Discipline

Status: Pending

Dependencies:

TASK-0003

Goal:

Align runtime/platform naming on-node with the Aluna umbrella and Aluna Context workload naming without changing validated storage/RAID identity.

Scope:

• verify and normalize hostname usage to `aluna-node-01` in runbooks and operational references  
• confirm namespace naming policy uses `aluna-*` conventions consistently  
• audit service/config references for remaining legacy `opencontext-*` or `alethos*` runtime names and document migration steps  
• produce explicit migration guidance for any rename that is operationally sensitive

Non-scope:

• destructive storage identity changes (UUID/mount source rewrites)  
• relabeling ext4 volume `alethos-data` without approved migration + rollback plan

Acceptance Criteria:

• runtime naming policy is documented and consistent for active namespaces/services  
• any remaining legacy identifiers are tracked as explicit follow-up actions with rollback notes  
• no regression to TASK-0001/TASK-0002 validated storage state

Verification:

kubectl get ns  
kubectl get all -A  
rg "opencontext|alethos" docs tasks context