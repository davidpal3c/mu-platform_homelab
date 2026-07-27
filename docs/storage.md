# Storage architecture

**Status: BUILT.** All four tiers are implemented on `mu-node-01`, mounted at boot by UUID, and verified after reboot.

The storage layer is where the "single node, built like a cluster" idea actually shows up. One host, four physical devices, one job each — so that OS churn, container churn, database IO, and backups cannot damage one another.

---

## 1. Tier map

| Tier | Device | ~Capacity | Filesystem | Mount | Purpose |
|------|--------|-----------|------------|-------|---------|
| Boot | `sda` + `sdc` (Intel SATA SSD) | 2 × ~180 GB | ext4 on mdadm RAID1 | `/boot`, `/`, `/var` | OS, system services, k3s binaries |
| Platform | `nvme1n1` (WD SN530) | ~238 GB | ext4 | `/platform` | Container runtime, kubelet, Prometheus, Loki |
| Database | `nvme0n1` (WD SN530) | ~477 GB | ext4 | `/data` | PostgreSQL, WAL, Redis |
| Backup | `sdb` (SATA HDD) | ~931 GB | ext4 | `/backups` | Dumps, snapshots, archives |

Two details worth stating plainly, because both have bitten this project before:

- **The larger NVMe is the database tier**, not the platform tier. Smaller NVMe (~238 GB) → `/platform`; larger (~477 GB) → `/data`.
- **The backup device is `sdb`, not `sdc`.** `sdc` is the second RAID1 boot SSD. Confusing the two would destroy the OS mirror. Every destructive procedure in this repo re-verifies device identity with `lsblk -o NAME,SIZE,MODEL` before touching anything.

---

## 2. Boot tier

```
sda ─┐
     ├─ mdadm RAID1 ─┬─ md0 → /boot
sdc ─┘               ├─ md1 → /
                     └─ md2 → /var
```

- **Boot mode:** UEFI.
- **ESP:** `sda1` mounted at `/boot/efi`; `sdc1` is a synced secondary ESP.
- **GRUB:** installed on both `/dev/sda` and `/dev/sdc`.
- **`/var` is a separate array member** — the classic failure mode is logs or container data filling the root filesystem and bricking the OS. Isolating `/var` bounds that, and the bind mounts below move the worst offenders off this tier entirely.

**Failure behaviour:** one boot SSD can fail and the node stays up, bootable from the survivor. Note the honest caveat — a **manual failover boot test is still outstanding**. Until it is exercised, RAID1 here is a design property, not a proven one.

**Operational follow-up:** the secondary ESP must be re-synced after kernel or GRUB updates. It does not update itself.

---

## 3. Platform tier — `/platform`

The write-heavy tier. Container image layers, kubelet state, and observability retention all churn hard, and none of it belongs on the boot mirror.

Five bind mounts redirect the high-churn paths:

| Source | Target |
|--------|--------|
| `/platform/rancher` | `/var/lib/rancher` |
| `/platform/kubelet` | `/var/lib/kubelet` |
| `/platform/containerd` | `/var/lib/containerd` |
| `/platform/prometheus` | `/var/lib/prometheus` |
| `/platform/loki` | `/var/lib/loki` |

Bind mounts rather than symlinks, because k3s and containerd both behave better against real mountpoints, and because `findmnt` then gives an unambiguous answer to "is this actually landing on the platform disk?"

**`fstab` ordering matters:** the `/platform` filesystem line must appear *before* its bind lines, or the binds mount onto an empty directory. Base mount before binds — every time.

**Failure behaviour:** if `/platform` fills or fails to mount, the OS and database tier are untouched. Workloads are affected; the node stays recoverable.

---

## 4. Database tier — `/data`

| Directory | Contents |
|-----------|----------|
| `/data/postgres` | PostgreSQL data directory |
| `/data/postgres_wal` | Write-ahead log — logically isolated mountpoint |
| `/data/redis` | Redis persistence |

Latency-sensitive, steady-write IO, deliberately kept away from Prometheus and Loki retention growth. WAL gets its own mountpoint even though it currently shares the physical NVMe — it enforces the operational habit now, and turns a future physical split into a config change instead of a migration.

**Capacity planning uses the measured ~477 GB.** This device is 512 GB-class hardware; earlier drafts of this project called it "1 TB" and that number was wrong. Plan against what `lsblk` reports, not against the label on the box.

**As-built note:** the ext4 volume label on this device is `alethos-data`, set when the filesystem was created under the project's former name. It is recorded here because it is what `blkid` reports on the live node, and it stays recorded that way until the disk itself changes — documentation that runs ahead of the hardware is worse than documentation that admits an inconsistency.

Relabelling to `mu-data` is **scheduled**, not deferred. `tune2fs -L` rewrites one field in the ext4 superblock: the UUID is untouched, every `fstab` line keys on `UUID=`, and nothing on this node resolves the device through `/dev/disk/by-label/`. The tier currently holds no database, so the change is being made while the blast radius is an empty filesystem rather than after Postgres owns it.

---

## 5. Backup tier — `/backups`

Cheap capacity on spinning disk, physically separate from the data it protects. Large sequential writes, low IOPS requirement, and no competition with hot-tier IO.

Scope, retention, restore verification, and off-host replication are covered in [backups.md](backups.md) — currently **planned**, not built. A backup device with no verified restore is storage, not a backup.

---

## 6. Persistence and mount options

All tier filesystems are mounted from `/etc/fstab` by **`UUID=`** — never by `/dev/sdX`, which reorders across reboots and kernel upgrades.

| Line type | Options |
|-----------|---------|
| Tier filesystems | `defaults,nofail` |
| Bind mounts | `bind,nofail` |

**Why `nofail`:** a missing or failed disk degrades the node instead of dropping boot into an emergency shell — the right trade-off for a headless machine reachable only over SSH.

**What `nofail` costs:** services can start against an empty directory when a tier silently fails to mount. That is a real risk and it is not solved by removing the flag; it is solved by alerting on expected mountpoints, which is a required deliverable of the observability phase.

---

## 7. Verification

The standard check set after any change touching storage:

```bash
lsblk -f                       # filesystems, labels, UUIDs
findmnt                        # what is actually mounted where
findmnt /var/lib/rancher \
        /var/lib/kubelet \
        /var/lib/containerd    # confirm binds resolve to /platform
df -h                          # capacity per tier
cat /proc/mdstat               # RAID1 state — expect [UU]
grep -v '^#' /etc/fstab        # base-before-bind ordering
```

A reboot test is part of acceptance, not an optional extra. Mounts that work until the next reboot are not persistent.

---

## 8. Known follow-ups

| Item | Status |
|------|--------|
| Manual disk-failover boot test | Outstanding — RAID1 unproven until exercised |
| Secondary ESP re-sync after kernel/GRUB updates | Recurring operational step |
| Per-mount usage alerting (80 / 90 / 95 %) | Blocked on observability phase |
| SMART health and temperature monitoring | Blocked on observability phase |
| Prometheus / Loki retention limits on `/platform` | To be set when observability is deployed |
| ext4 relabel `alethos-data` → `mu-data` on `/data` | Scheduled — bundled with the host rename into one change window |
| Host rename to `mu-node-01`, incl. mdadm homehost re-stamp | Scheduled — must land before k3s bootstrap |
| Off-host backup replication | Design pending — see [backups.md](backups.md) |
