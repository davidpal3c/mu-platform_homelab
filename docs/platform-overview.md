# Platform Overview

**Purpose:** High-level view of the **Aluna Platform** homelab **storage and boot** layout, tier separation, and operational model. Umbrella ontology and platform diagrams: [aluna-platform/](aluna-platform/README.md). **Aluna Context** workload architecture: [aluna-context/](aluna-context/README.md).

**Last updated:** 2026-03-28 (verified against `docs/build-reports/reviews/REVIEW-TASK-0002.md` **PASS**; as-built sizes and device names match `BR-TASK-0002` / `context/project-context.json`)

**Storage reality (post TASK-0002):** All four tiers are **mounted at boot** (UUID `fstab`, `nofail`); high-churn paths are **bind-mounted** from `/platform`. Smaller NVMe is **`nvme1n1` → `/platform`**; larger is **`nvme0n1` → `/data`** (~477G, 512G-class—not nominal 1TB).

---

## 1. What the platform is

The Aluna Platform runs on a **single-node homelab** (Lenovo ThinkStation P510, **aluna-node-01**) but is designed like a **mini production cluster**: storage and IO are separated by tier so failure domains are clear and blast radius is minimized.

- **OS and boot** — RAID1 on two Intel ~180GB SATA SSDs (`sda` + `sdc`).
- **Platform workloads** (future k3s + observability data) — smaller NVMe **`nvme1n1`** (~238G) at **`/platform`**, with **bind mounts** so `/var/lib/{rancher,kubelet,containerd,prometheus,loki}` use platform disk, not the OS RAID.
- **Databases** — larger NVMe **`nvme0n1`** (~477G, 512G-class WD SN530, **not** a literal 1TB on this hardware) at **`/data`**.
- **Backups** — **1TB-class SATA HDD `sdb`** at **`/backups`** (never use **`sdc`** for backups — it is the second RAID SSD).

This separation keeps high-churn runtime IO off the boot tier and off database disks.

---

## 2. Storage and boot layout (current state)

### Implemented

| Tier | Device (this node) | ~Capacity | Mount | Notes |
|------|--------------------|-----------|-------|--------|
| Boot RAID | `sda` + `sdc` | 2× ~180G | `/boot`, `/`, `/var` + ESP | md0/md1/md2; TASK-0001 |
| Platform | `nvme1n1` | ~238G | `/platform` | ext4, UUID fstab; five binds to `/var/lib/*`; TASK-0002 |
| Database | `nvme0n1` | ~477G | `/data` | ext4; postgres / WAL / redis dirs; TASK-0002 |
| Backup | `sdb` | ~931G | `/backups` | ext4; HDD; TASK-0002 |

**Persistence:** Tier filesystem lines in `/etc/fstab` use **`UUID=`** and **`defaults,nofail`**. Bind mounts use **`bind,nofail`** and appear **after** the `/platform` line.

**Not yet deployed:** k3s (TASK-0003), observability stack, Aluna Context workloads.

---

## 3. Failure domains and operational expectations

| Tier | Failure domain | Operational expectation |
|------|----------------|-------------------------|
| Boot | Single node; RAID1 | One SSD failure: degraded RAID but bootable from other disk if firmware/ESP OK. |
| Platform | One NVMe | If full or missing (`nofail`): OS/DB may still boot; fix mount before relying on k3s. |
| Database | One NVMe | Isolated from platform churn; capacity planning uses **~477G** on this node, not “1TB”. |
| Backup | One HDD | Disk loss: restore from off-host copy when policy exists. |

Disk checks: `lsblk -f`, `findmnt`, `df -h`, `cat /proc/mdstat`. Full operator checklist: `docs/build-reports/BR-TASK-0002.md` Part C.7.

---

## 4. References

- **Architecture detail:** `docs/system-blueprint.md`
- **Live layout summary:** `context/Final_disk_layout_all_drives.md`
- **Structured context:** `context/project-context.json`
- **Implementation:** `docs/build-reports/BR-TASK-0002.md`
- **Verification:** `docs/build-reports/reviews/REVIEW-TASK-0002.md`
