# Disk layout — aluna-node-01 (canonical, post TASK-0002)

This file is the **authoritative summary** of the **live** storage layout after **TASK-0001** (boot RAID) and **TASK-0002** (platform / database / backup mounts + binds).  
Detailed runbook: `docs/build-reports/BR-TASK-0002.md`.  
Structured context: `context/project-context.json`.

**Danger:** Older drafts in version control may have shown RAID on `sda`+`sdb` or backups on `sdc`. **On this node that is wrong** — see the table below before any `mkfs` or partition work.

---

## Live device roles (aluna-node-01)

| Role | Devices | Mount(s) | Notes |
|------|---------|----------|--------|
| Boot RAID1 | **`/dev/sda`** + **`/dev/sdc`** (Intel SSDs) | `/boot/efi` (sda1), `/boot` (md0), `/` (md1), `/var` (md2) | **Never** format `sda`/`sdc`/`md*` for tier work. |
| Platform | **`/dev/nvme1n1`** (~238G SN530) | `/platform` | ext4, label `platform`. May be whole-disk ext4 (no `p1`) — UUID in fstab must match `blkid`. |
| Database | **`/dev/nvme0n1`** (~477G SN530, 512G-class) | `/data` | ext4, label `alethos-data`. Smaller NVMe is **not** DB on this node — larger is DB. |
| Backup | **`/dev/sdb`** (~931G HDD) | `/backups` | ext4, label `backups`. **`sdc` is RAID — not backup.** |

---

## UUIDs (`/etc/fstab` — confirm on node with `blkid`)

| Mount | UUID | Label |
|-------|------|--------|
| `/platform` | `70a566e7-4834-48a3-8b5a-a97fc0383488` | `platform` |
| `/data` | `8650424b-1fff-4f16-80a8-43b8545f27ba` | `alethos-data` |
| `/backups` | `bded6290-6ef8-4c02-90a2-0fff87394ba0` | `backups` |

Boot lines may use installer `by-id`; tier lines should use **`UUID=`** only.

---

## Bind mounts (require `/platform` mounted first)

| Source | Target |
|--------|--------|
| `/platform/rancher` | `/var/lib/rancher` |
| `/platform/kubelet` | `/var/lib/kubelet` |
| `/platform/containerd` | `/var/lib/containerd` |
| `/platform/prometheus` | `/var/lib/prometheus` |
| `/platform/loki` | `/var/lib/loki` |

Options: `bind,nofail` in `/etc/fstab`.

---

## Directory layout

**Under `/data`:** `postgres`, `postgres_wal`, `redis`  
**Under `/platform`:** `rancher`, `kubelet`, `containerd`, `prometheus`, `loki`  
**Under `/backups`:** operator-defined dumps/archives (no hot DB data)

---

## Boot tier detail (RAID + EFI)

Per **boot** disk (`sda` and `sdc` only):

- `p1`: 512MiB ESP (FAT32) — primary mount `sda1` → `/boot/efi`; `sdc1` secondary ESP  
- `p2`: 2GiB → md0 → `/boot`  
- `p3`: 50GiB → md1 → `/`  
- `p4`: remainder → md2 → `/var`  

`grub-install` on **both** `/dev/sda` and `/dev/sdc`. Sync secondary ESP after kernel/GRUB updates (`tasks/lessons.md` — INF-006).

---

## Visual summary

```
  RAID1 BOOT (sda + sdc)          nvme1n1          nvme0n1           sdb
  md0 /boot  md1 /  md2 /var  →  /platform    →   /data         →   /backups
                                      │               │
                                      └── bind ──→ /var/lib/{rancher,kubelet,containerd,prometheus,loki}
```

---

## Operational checks

```bash
lsblk -f
grep -v '^#' /etc/fstab | grep -v '^$'
findmnt -A | grep -E 'platform|/data|backups|rancher|kubelet|containerd|prometheus|loki'
df -h /platform /data /backups
cat /proc/mdstat
```

---

## Legacy / superseded content

Older planning notes that assumed:

- RAID mirror on `sda` + **`sdb`**, or  
- Backup tier on **`/dev/sdc1`**

are **obsolete for aluna-node-01**. Always use **`lsblk -o NAME,SIZE,MODEL`** before destructive operations.
