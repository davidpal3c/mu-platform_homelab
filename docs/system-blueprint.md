# System Blueprint

**Purpose:** Current architecture truth for the Aluna Platform. Describes what exists, how it fits together, trust boundaries, and deployment model so that any future agent or operator can reconstruct the system.

**Last updated:** 2026-03-28 (verified against `docs/build-reports/reviews/REVIEW-TASK-0002.md` **PASS** and `docs/build-reports/BR-TASK-0002.md` live as-built record)

**Storage reality:** TASK-0002 moved the node from “boot RAID only; NVMe + HDD unused” to **persistent `/platform`, `/data`, `/backups`** plus **five bind mounts** from `/platform/*` to `/var/lib/*`. Boot RAID (`sda`+`sdc`, md0–md2) was not reformatted. See review report §“Storage reality change”.

---

## 1. Platform identity

- **Product platform:** **Aluna** (ontology: **Ceiba**, Aluna Context, Aluna Mira, Aluna Terra — see [docs/aluna-platform/platform-ontology.md](aluna-platform/platform-ontology.md)). **Aluna Context** runtime architecture: [docs/aluna-context/architecture-diagrams/](aluna-context/architecture-diagrams/architecture%20diagrams.md).
- **Machine:** Lenovo ThinkStation P510 (Xeon E5-2640 v4, 32GB RAM).
- **Node:** aluna-node-01 (homelab; align `hostname` on the host when renaming).
- **OS:** Ubuntu Server LTS (headless, SSH-accessible).
- **Kubernetes:** k3s (not yet deployed; Phase 0 — next: TASK-0003).
- **Role:** Single-node homelab designed as a "mini production cluster" via storage-tier and failure-domain separation.

---

## 2. Storage architecture

All **four tiers** are implemented on aluna-node-01 (TASK-0001 boot + TASK-0002 platform/database/backup). Boot RAID was **not** reformatted for TASK-0002.

### 2.1 Boot tier (implemented — TASK-0001)

**Hardware mapping:**

| Device | Model | Role |
|--------|--------|------|
| `/dev/sda` | Intel SSD Pro 1500 (~180GB) | Boot SSD 1 |
| `/dev/sdc` | Intel SSD 520 (~180GB) | Boot SSD 2 |
| `/dev/sdb` | WDC WD10SPZX ~931G HDD | **Backup tier** — `/backups` (not part of RAID) |

**Layout:**

- **Boot mode:** UEFI only.
- **RAID:** mdadm RAID1 on `sda` + `sdc` only.
- **Arrays:** `md0` → `/boot`, `md1` → `/`, `md2` → `/var`.
- **EFI:** `sda1` @ `/boot/efi`; `sdc1` secondary ESP (synced / bootable).
- **GRUB:** On both `/dev/sda` and `/dev/sdc`.

**Failure domain:** Single node; one boot SSD failure tolerated by RAID1.

### 2.2 Platform tier (implemented — TASK-0002)

| Field | Value |
|-------|--------|
| **Block device** | `/dev/nvme1n1` (smaller NVMe on this node, WD SN530 ~238.5G) |
| **Filesystem** | ext4, label `platform` |
| **Mount** | `/platform` |
| **fstab** | `UUID=70a566e7-4834-48a3-8b5a-a97fc0383488` … `defaults,nofail` |

**Layout note:** Live system may show ext4 on the **whole disk node** (no `nvme1n1p1`); UUID in `fstab` must match `blkid` for the node actually mounted.

**Subdirectories:** `rancher`, `kubelet`, `containerd`, `prometheus`, `loki`.

**Bind mounts** (`/etc/fstab`, **after** `/platform` line; `bind,nofail`):

| Source | Target |
|--------|--------|
| `/platform/rancher` | `/var/lib/rancher` |
| `/platform/kubelet` | `/var/lib/kubelet` |
| `/platform/containerd` | `/var/lib/containerd` |
| `/platform/prometheus` | `/var/lib/prometheus` |
| `/platform/loki` | `/var/lib/loki` |

**Role:** k3s runtime and observability persistence; high churn kept off boot RAID and `/var` on md2.

### 2.3 Database tier (implemented — TASK-0002)

| Field | Value |
|-------|--------|
| **Block device** | `/dev/nvme0n1` (larger NVMe on this node, WD SN530 ~476.9G — **512G-class, not literal 1TB**) |
| **Filesystem** | ext4, volume label `alethos-data` (set at TASK-0002 `mkfs`; matches live `blkid` — Aluna **Context** data plane; optional relabel to `aluna-data` only with care) |
| **Mount** | `/data` |
| **fstab** | `UUID=8650424b-1fff-4f16-80a8-43b8545f27ba` … `defaults,nofail` |

**Directories:** `/data/postgres`, `/data/postgres_wal`, `/data/redis`.

**Role:** PostgreSQL/PostGIS, Redis; isolated from platform IO.

**Sizing decision:** Smaller NVMe → `/platform`, larger → `/data` (documented in `docs/build-reports/BR-TASK-0002.md`).

### 2.4 Backup tier (implemented — TASK-0002)

| Field | Value |
|-------|--------|
| **Block device** | `/dev/sdb` (SATA HDD ~931.5G) |
| **Filesystem** | ext4, label `backups` |
| **Mount** | `/backups` |
| **fstab** | `UUID=bded6290-6ef8-4c02-90a2-0fff87394ba0` … `defaults,nofail` |

**Role:** Dumps, archives, retention — **not** `sdc` (RAID member).

**Critical:** Always confirm HDD identity with `lsblk -o NAME,SIZE,MODEL` before any `mkfs` on backup tier (`tasks/lessons.md` — INF-007).

---

## 3. Trust and operational boundaries

- **Boot tier:** OS; RAID1 + dual ESP.
- **Platform / database / backup:** Separate mount trees; `nofail` on tier lines avoids total boot hang if a disk is missing — services may start without data until fixed (homelab tradeoff; add alerting when observability lands).
- **Bind order:** Base `/platform` before bind lines in `fstab`.
- **Observability:** Disk layout verifiable via `lsblk -f`, `findmnt`, `df -h`, `cat /proc/mdstat`. Metrics/alerts deferred to later phases.

---

## 4. Runtime and deployment model (current)

- **Node:** Single physical host (aluna-node-01).
- **Services:** Ubuntu Server base, OpenSSH, base tooling; storage tiers mounted at boot. **k3s not installed** until TASK-0003.
- **Network:** DHCP on primary NIC (e.g. eno1); static IP may follow.
- **Security baseline:** SSH enabled; hardening (keys, firewall) in later tasks.

---

## 5. References

- **Context:** `context/project-context.json`
- **Disk layout (canonical):** `context/Final_disk_layout_all_drives.md`
- **Build / runbook:** `docs/build-reports/BR-TASK-0002.md`
- **Review:** `docs/build-reports/reviews/REVIEW-TASK-0002.md`
- **Boot task:** `docs/build-reports/TASK-0001.md`, `docs/build-reports/reviews/REVIEW-TASK-0001.md`
