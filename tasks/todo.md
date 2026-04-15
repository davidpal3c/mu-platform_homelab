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

**Canonical spec:** [`tasks/TASK_0003.md`](TASK_0003.md) — this section is the queue index only (goal, status, dependencies, doc pointers).

Status: Ready

Dependencies:

TASK-0002 (complete)

Why now:

Phase 0B storage is complete and verified. k3s can install with runtime paths on `/platform` bind mounts. Phase 1 TASK-0004 (observability) and ingress work stay gated until TASK-0003 is accepted.

Goal:

Single-node k3s on **aluna-node-01**; baseline namespaces **`aluna-access`**, **`aluna-context`**, **`aluna-mira`**, **`aluna-terra`**, **`observability`** only (no new `opencontext-*`). Full scope, lessons preamble, acceptance criteria, and Builder mode: **`TASK_0003.md`**.

Non-scope (summary):

TLS (TASK-0005); observability stack install (TASK-0004); Aluna Context API / workloads.

Risks (summary):

Wrong-path / missing binds; single-node resource pressure; default exposure surface — see spec.

Files / docs affected:

`platform/k3s/cluster-setup.md`  
`docs/system-blueprint.md`  
`docs/platform-overview.md`  
`docs/build-reports/BR-TASK-0003.md` (after operator sign-off per `TASK_0003.md`)  
`context/project-context.json`

Acceptance / verification / Builder output:

Per **`tasks/TASK_0003.md`** (includes deferred build report until operator confirms).

---

# TASK-0004

Title: Observability stack baseline

Phase: Phase 1 — Observability Discipline

Status: Pending

Dependencies:

TASK-0003 (must be **Complete** / accepted — cluster Ready, namespaces baseline)

Why now:

Roadmap requires the platform to observe itself before meaningful workloads. k3s must exist first so components run as cluster workloads and scrape node/kube metrics.

Goal:

Deploy Prometheus, Grafana, Loki, and node-exporter (and Alertmanager when in scope) so disk-per-tier and node health are visible.

Scope:

• Prometheus + Grafana + Loki (+ node-exporter; Alertmanager per roadmap Phase 1C when scoped)  
• Dashboards including **disk usage per mount** (`/`, `/var`, `/platform`, `/data`, `/backups`)  
• Namespaces aligned with Aluna model (e.g. metrics in `observability` or as spec’d later)

Non-scope:

• Aluna Context API deploy  
• Production on-call runbooks (can stub)  
• TLS for Grafana via public ingress (may use ClusterIP / port-forward until TASK-0005)

Risks:

• `/platform` fill from Prometheus/Loki retention — plan limits in spec when executing.  
• Single-node CPU/memory contention with k3s control plane.

Files / docs affected:

`docs/system-blueprint.md`  
`docs/platform-overview.md`  
`docs/build-reports/BR-TASK-0004.md` (or name decided at execution)  
`context/project-context.json`

Acceptance Criteria:

• Metrics visible in Grafana; **disk usage per tier** visible  
• Container / node logs ingested in Loki  
• Core probes documented in build report

Verification:

Grafana UI; Prometheus targets up; Loki query returns recent lines; `kubectl get pods -n observability` (or chosen ns) healthy.

Builder output:

Operator runbook + build report after operator confirmation (same discipline as TASK-0003 unless delegated).

---

# TASK-0005

Title: Ingress controller + TLS

Phase: Phase 2 — Deployment Discipline

Status: Pending

Dependencies:

TASK-0003 (cluster baseline). TASK-0004 recommended before exposing anything externally (observability-first).

Why now:

Public HTTPS path is a Phase 2 capability; depends on a working cluster and sane internal routing.

Goal:

Ingress controller, TLS, and domain mapping for future Aluna-facing endpoints.

Scope:

• Ingress controller install (choice documented: e.g. Traefik default vs nginx — pick one in runbook)  
• TLS certificates (ACME / cert-manager or k3s defaults — to be fixed in spec at execution)  
• Domain routing smoke test

Non-scope:

• Aluna Context API container deploy (later phase)  
• Full zero-trust / WAF — document follow-ups only

Risks:

• Accidental exposure of admin UIs or kube API — bind to internal interfaces until hardened (SEC-001).  
• Certificate rate limits / DNS prerequisites.

Files / docs affected:

`docs/system-blueprint.md`  
`docs/platform-overview.md`  
`docs/build-reports/BR-TASK-0005.md`  
`context/project-context.json`

Acceptance Criteria:

• `https://<domain>/...` health or test route succeeds  
• Ingress routes to a **test** service only (no production workload required)

Verification:

`curl -fsS https://.../health` (or equivalent); `kubectl get ingress -A`

Builder output:

Operator runbook + build report after operator confirmation.

---

# TASK-0006

Title: Runtime naming convergence (post-bootstrap, non-destructive)

Phase: Phase 1 — Observability Discipline (may start after TASK-0003; **does not** unblock TASK-0004)

Status: Pending

Dependencies:

TASK-0003 (cluster exists so `kubectl` / manifest names can be audited)

Why now:

Umbrella rebrand to **Aluna** leaves historical identifiers (`alethos-data` volume label, `opencontext-*` in old diagrams, git remote name). Convergence is documentation + planning unless a dedicated migration task is opened.

Goal:

Align **documentation and operational references** with Aluna naming; track sensitive renames without touching validated storage layout.

Scope:

• Hostname `aluna-node-01` consistent in runbooks and context  
• Namespace policy: **`aluna-*` + `observability`** as baseline; no new `opencontext-*` without ADR  
• Audit `docs/`, `tasks/`, `context/` for legacy strings; output a **migration matrix** (what / risk / rollback)  
• **As-built:** ext4 label **`alethos-data`** on `/data` remains valid until a **separate** migration task — do not “fix” completed BR/review PDFs for branding alone

Non-scope:

• Destructive storage identity changes (UUID, `fstab` source swaps)  
• **`tune2fs -L`** relabel `alethos-data` → `aluna-data` without approved migration + rollback task  
• Rewriting historical build/review artifacts without operator intent

Risks:

• “Cosmetic” edits that imply storage changed when it did not — keep forward-only distinction clear.  
• Partial rename in k8s manifests breaking Helm releases — execute only with rollback.

Files / docs affected:

`docs/system-blueprint.md`  
`docs/platform-overview.md`  
`docs/aluna-platform/*`, `docs/aluna-context/*` (cross-links)  
`context/project-context.json`  
`tasks/backlog.md` (follow-ups)  
`docs/build-reports/BR-TASK-0006.md` (naming audit report — create at execution)

Acceptance Criteria:

• Naming policy doc or section added/updated  
• Legacy identifier list + **non-destructive** next steps  
• TASK-0001/TASK-0002 storage verification re-run shows **no drift** from PASS state

Verification:

`kubectl get ns` ; `rg -n "opencontext|alethos" docs tasks context` ; spot-check `blkid` / `findmnt /data` unchanged from TASK-0002 record

Builder output:

Audit report in repo + operator checklist (no cluster changes unless explicitly scoped in a sub-step).