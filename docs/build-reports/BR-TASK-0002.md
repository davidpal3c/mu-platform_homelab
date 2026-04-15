BR-TASK-0002 — Storage tier mount configuration (Phase 0B)
==========================================================

**Single source of truth:** This file is the **only** build-report + operator runbook for TASK-0002 under `docs/build-reports/`. It records **what was implemented** on aluna-node-01 and the **repeatable procedure** for the same class of work.

**Canonical task spec (scope, acceptance):** `tasks/TASK_0002.md`  
**Node:** aluna-node-01  
**Dependency:** TASK-0001 complete (RAID1 boot tier live).  
**Build report naming:** Use `BR-TASK-XXXX.md` for formal build reports (`context/AGENTS.md` workflow). Do not modify `docs/build-reports/TASK-0001.md` as part of this task.

---

Part A — Task reference (from Planner)
--------------------------------------

**Objective:** Implement platform NVMe, database NVMe, and backup HDD: partitions (or equivalent), ext4, **UUID-based** `/etc/fstab`, directory layout, bind mounts for k3s/observability paths. **Do not** reformat boot RAID.

**Non-scope:** k3s, observability stack.

**Acceptance (summary):** Reboot persists `/platform`, `/data`, `/backups` and bind mounts; UUID-only lines for those filesystems in `fstab`; RAID unchanged and healthy. Full checklist: `tasks/TASK_0002.md`.

---

Part B — Build report (implementation record — aluna-node-01)
---------------------------------------------------------------

### B.1 Goal (implemented)

- **Platform:** `/platform` on smaller NVMe, bind mounts to `/var/lib/{rancher,kubelet,containerd,prometheus,loki}`.
- **Database:** `/data` with `postgres`, `postgres_wal`, `redis`.
- **Backup:** `/backups` on **SATA HDD `sdb`** (not `sdc`).
- **Constraints:** Tier lines in `fstab` use **`UUID=`**; boot RAID (`sda`+`sdc` → md0/md1/md2) untouched.

### B.2 Context summary

- Phase 0B — storage architecture (`context/roadmap.md`).
- Legacy docs sometimes placed backup on `sdc1`; **on this node `sdc` is the second Intel RAID SSD.** Backup tier is **`/dev/sdb`**.

### B.3 Configuration implemented (live state)

| Role | Block device | Model / size (approx.) | Filesystem | Mount |
|------|--------------|-------------------------|------------|--------|
| Boot RAID | `sda` + `sdc` | Intel Pro 1500 + Intel 520 | md0/md1/md2 ext4 | `/boot`, `/`, `/var` |
| Platform | `/dev/nvme1n1` | WD SN530 ~238.5G | ext4, label `platform` | `/platform` |
| Database | `/dev/nvme0n1` | WD SN530 ~476.9G (512G-class) | ext4, label `alethos-data` | `/data` |
| Backup | `/dev/sdb` | WDC WD10SPZX ~931.5G | ext4, label `backups` | `/backups` |

**Layout nuance:** `lsblk -f` shows **ext4 on the whole block device** for `nvme0n1`, `nvme1n1`, and `sdb` (no `p1` children in the live tree). `findmnt` reports `SOURCE` as those disk nodes. **Valid** — `fstab` UUIDs match `blkid` for those devices.

**UUIDs (`fstab`) — confirm on node with `blkid`:**

| Mount | UUID | Label |
|-------|------|--------|
| `/platform` | `70a566e7-4834-48a3-8b5a-a97fc0383488` | `platform` |
| `/data` | `8650424b-1fff-4f16-80a8-43b8545f27ba` | `alethos-data` |
| `/backups` | `bded6290-6ef8-4c02-90a2-0fff87394ba0` | `backups` |

**Bind mounts (`fstab`, after `/platform`):**

| Source | Target |
|--------|--------|
| `/platform/rancher` | `/var/lib/rancher` |
| `/platform/kubelet` | `/var/lib/kubelet` |
| `/platform/containerd` | `/var/lib/containerd` |
| `/platform/prometheus` | `/var/lib/prometheus` |
| `/platform/loki` | `/var/lib/loki` |

Options: `defaults,nofail` on tier ext4; `bind,nofail` on binds.

**RAID:** `md0`/`md1`/`md2` remain **active raid1**, **`[2/2] [UU]`**.

### B.4 Decisions made

- Smaller NVMe → `/platform`, larger → `/data` (docs may say “1TB data”; hardware here is ~477G — document in `project-context.json`).
- Backup on **`sdb`**, not `sdc`.
- ext4 for all three tiers; ZFS deferred.
- Whole-disk ext4 outcome accepted; repartitioning later would require migration.

### B.5 Commands executed (reference)

```bash
lsblk -o NAME,SIZE,MODEL,TYPE
lsblk -f
cat /proc/mdstat

export PLAT_DEV=/dev/nvme1n1
export DATA_DEV=/dev/nvme0n1
export BACKUP_DEV=/dev/sdb

# parted: mklabel gpt + mkpart on each tier disk (see Part C)
# mkfs.ext4 -L ... on partition or whole disk per lsblk outcome
# blkid → fstab UUID lines + bind lines
sudo cp -a /etc/fstab /etc/fstab.pre-task-0002
sudo systemctl daemon-reload
sudo mount -a -v
```

### B.6 Validation (post-reboot — met)

- `findmnt /platform` / `/data` / `/backups` show expected sources.
- `mount -a -v` succeeds; all five `/var/lib/*` binds present.
- `lsblk -f` UUIDs match `fstab`; `nvme1n1` shows `/platform` + bind mountpoints.
- `cat /proc/mdstat` — all `[UU]`.

### B.7 Files and systems touched

| Location | Change |
|----------|--------|
| `nvme1n1`, `nvme0n1`, `sdb` | ext4 tier filesystems |
| `/etc/fstab` | UUID + bind entries |
| `/platform`, `/data`, `/backups`, `/var/lib/*` | directories / bind targets |
| `sda`/`sdc`/md* | **unchanged** |

**Repo:** this file only (no app code).

### B.8 Assumptions

- Operator verified devices with `lsblk` before `mkfs`.
- Curtin `fstab` lines for `/` on md may remain `by-id`; tier lines use `UUID=`.

### B.9 Summary for Reviewer-Verifier

- **Changed:** Three tiers + five binds; RAID healthy; no k3s.
- **Gap:** Update blueprint/context for real NVMe sizes and `sdb`/`sdc` roles.
- **Next:** TASK-0003 after TASK-0002 closed in `tasks/todo.md`.

---

Part C — Operator runbook (repeatable procedure)
------------------------------------------------

Use **before any destructive command:** `lsblk -f` and confirm **boot = `sda`+`sdc`**, **backup HDD = `sdb`**, **two NVMe** — never `mkfs` on `sda`, `sdc`, or `md*`.

### C.0 Preconditions

- Sudo user; TASK-0001 verified.
- Backup data on `sdb` if needed — proceeding **wipes** tier disks you format.

### C.1 Identify disks

```bash
lsblk -o NAME,SIZE,MODEL,TYPE
lsblk -f
cat /proc/mdstat
```

Assign:

- **`$PLAT_DEV`** — ~238–240G NVMe (e.g. `/dev/nvme1n1`).
- **`$DATA_DEV`** — larger NVMe (e.g. `/dev/nvme0n1`).
- **`$BACKUP_DEV=/dev/sdb`** — ~931G HDD.

```bash
export PLAT_DEV=/dev/nvme1n1
export DATA_DEV=/dev/nvme0n1
export BACKUP_DEV=/dev/sdb
```

**Stop** if `PLAT_DEV` or `DATA_DEV` would be `sda`, `sdc`, or `md*`.

### C.2 Partition each tier disk (GPT + one partition)

**Destructive.** If you already have a satisfactory single partition, skip to C.3 using `${DEV}p1` or whole-disk per `lsblk`.

```bash
sudo wipefs -a "$PLAT_DEV" 2>/dev/null || true
sudo parted -s "$PLAT_DEV" mklabel gpt
sudo parted -s "$PLAT_DEV" mkpart platform ext4 1MiB 100%
sudo parted -s "$PLAT_DEV" set 1 name platform

sudo wipefs -a "$DATA_DEV" 2>/dev/null || true
sudo parted -s "$DATA_DEV" mklabel gpt
sudo parted -s "$DATA_DEV" mkpart data ext4 1MiB 100%
sudo parted -s "$DATA_DEV" set 1 name data

sudo wipefs -a "$BACKUP_DEV" 2>/dev/null || true
sudo parted -s "$BACKUP_DEV" mklabel gpt
sudo parted -s "$BACKUP_DEV" mkpart backups ext4 1MiB 100%
sudo parted -s "$BACKUP_DEV" set 1 name backups
```

```bash
lsblk -f "$PLAT_DEV" "$DATA_DEV" "$BACKUP_DEV"
```

**Format targets:** Use **`${DEV}p1`** for NVMe; **`${BACKUP_DEV}1`** (e.g. `/dev/sdb1`) for SATA. If `lsblk` later shows ext4 on the whole disk node with no `p1`, use **`blkid`** on that node for `fstab` — both patterns are valid.

### C.3 Create ext4 and capture UUIDs

```bash
sudo mkfs.ext4 -F -L platform     "${PLAT_DEV}p1"
sudo mkfs.ext4 -F -L alethos-data "${DATA_DEV}p1"
sudo mkfs.ext4 -F -L backups      "${BACKUP_DEV}1"
sudo blkid "${PLAT_DEV}p1" "${DATA_DEV}p1" "${BACKUP_DEV}1"
```

If you intentionally formatted a whole disk (no partition), run `sudo blkid "$PLAT_DEV"` (etc.) instead.

### C.4 Mount-once test

```bash
sudo mkdir -p /platform /data /backups
sudo mount UUID="<UUID_PLAT>" /platform
sudo mount UUID="<UUID_DATA>" /data
sudo mount UUID="<UUID_BAK>" /backups
df -h /platform /data /backups
sudo umount /backups /data /platform
```

### C.5 Directory layout

```bash
sudo mkdir -p /platform/{rancher,kubelet,containerd,prometheus,loki}
sudo mkdir -p /data/{postgres,postgres_wal,redis} /backups
sudo mkdir -p /var/lib/rancher /var/lib/kubelet /var/lib/containerd \
  /var/lib/prometheus /var/lib/loki
sudo chown root:root /platform /data /backups 2>/dev/null || true
sudo chmod 755 /platform /data /backups
```

### C.6 `/etc/fstab` (order: base mounts, then binds)

```bash
sudo cp -a /etc/fstab /etc/fstab.pre-task-0002
```

Append (your UUIDs):

```fstab
# TASK-0002 — storage tiers (UUID only)
UUID=<UUID_PLAT>  /platform  ext4  defaults,nofail  0  2
UUID=<UUID_DATA>  /data      ext4  defaults,nofail  0  2
UUID=<UUID_BAK>   /backups   ext4  defaults,nofail  0  2

# Bind mounts (require /platform mounted first)
/platform/rancher     /var/lib/rancher     none  bind,nofail  0  0
/platform/kubelet     /var/lib/kubelet     none  bind,nofail  0  0
/platform/containerd  /var/lib/containerd  none  bind,nofail  0  0
/platform/prometheus  /var/lib/prometheus  none  bind,nofail  0  0
/platform/loki        /var/lib/loki        none  bind,nofail  0  0
```

Optional: comment bind lines first, `mount -a`, verify base mounts, then uncomment binds.

```bash
sudo systemctl daemon-reload
sudo mount -a -v
```

### C.7 Verification checklist

```bash
lsblk -f
grep -v '^#' /etc/fstab | grep -v '^$'
findmnt -A | grep -E 'platform|/data|backups|rancher|kubelet|containerd|prometheus|loki'
df -h /platform /data /backups
cat /proc/mdstat
```

- [ ] Tier UUIDs in `fstab` match `blkid` for the **mounted** device nodes (may be whole disk or `p1`).
- [ ] All five binds show `/platform/...` as source.
- [ ] Capacities ~240G / larger NVMe / ~931G HDD.
- [ ] RAID `[UU]` on md0/md1/md2.

```bash
sudo reboot
```

After login: `findmnt /platform /data /backups` and `findmnt /var/lib/containerd`.

### C.8 Rollback

```bash
sudo umount /var/lib/loki /var/lib/prometheus /var/lib/containerd \
  /var/lib/kubelet /var/lib/rancher 2>/dev/null || true
sudo umount /backups /data /platform 2>/dev/null || true
sudo cp -a /etc/fstab.pre-task-0002 /etc/fstab
sudo mount -a
```

Boot failure: live USB, edit root `fstab` or comment TASK-0002 lines.

### C.9 Commands reference (summary)

| Step | Action |
|------|--------|
| Identify | `lsblk -f`, `lsblk -o NAME,SIZE,MODEL` |
| Partition | `parted` GPT + one partition per tier disk |
| Format | `mkfs.ext4 -L ...` on `p1` or whole disk per design |
| UUID | `blkid` |
| fstab | UUID lines then bind lines |
| Test | `mount -a`, `findmnt`, reboot |

---

Part D — Risks (consolidated)
-----------------------------

- **Wrong disk:** `mkfs` on `sda`/`sdc`/md destroys the OS — always `lsblk -f` first.
- **`sdb` vs `sdc`:** Backup is **`sdb`** on this node; **`sdc` is RAID.**
- **Doc vs hardware:** “1TB database NVMe” may be ~512G class — update context files.
- **Bind order / `nofail`:** Base `/platform` must be listed before binds; test boot behavior on your Ubuntu release.
- **Data loss:** `mkfs` erases prior content on tier disks.

---

Part E — Documentation and state updates (other agents)
-------------------------------------------------------

| File | Update |
|------|--------|
| `docs/system-blueprint.md` | Live four-tier layout, devices, bind map; optional UUIDs. |
| `docs/platform-overview.md` | Storage summary; real sizes (~240G / ~477G / ~931G). |
| `context/project-context.json` | `sdb` backup, `sdc` RAID, NVMe nodes, tiers implemented. |
| `context/Final_disk_layout_all_drives.md` | Align `sdc`/`sdb` and NVMe sizes or add “aluna-node-01 live” note. |
| `tasks/todo.md` | Mark TASK-0002 **Complete** after Reviewer + doc sync. |
| `tasks/lessons.md` | Optional: durable rules (e.g. verify HDD identity before backup `mkfs`). |

---

Part F — Follow-up
------------------

- **TASK-0003:** k3s bootstrap using bind-backed `/platform` paths.
- **Operator:** Keep `/etc/fstab.pre-task-0002` on node until sign-off.
