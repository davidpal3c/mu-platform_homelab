# Review Report — TASK-0002

**Task:** TASK-0002 — Storage tier mount configuration (Phase 0B)  
**Phase:** Phase 0 — Platform Foundation / 0B — Storage Architecture  
**Reviewed against:** `tasks/TASK_0002.md`, `docs/build-reports/BR-TASK-0002.md`, `tasks/lessons.md` (relevant entries)

---

## Status: **PASS**

Implementation described in **`docs/build-reports/BR-TASK-0002.md`** aligns with `tasks/TASK_0002.md` acceptance criteria. The build report records post-reboot validation (Part B.6). Documentation create/update is delegated to the Documentation Agent (Part E of the build report).

---

## Verification record (alethos-node-01 — per BR-TASK-0002)

The following is **as recorded** in Part B of the build report (operator should retain spot-check ability using §6).

### Storage reality change (this task vs post–TASK-0001)

**TASK-0002 materially changed on-disk and mount reality** on alethos-node-01. It was not documentation-only.

| Aspect | After TASK-0001 only | After TASK-0002 (implemented) |
|--------|---------------------|-------------------------------|
| **`nvme0n1` / `nvme1n1`** | Present but **not** used as persistent platform/database tiers (no `/platform` or `/data` from those devices). | **ext4**, **`/platform`** (smaller NVMe) and **`/data`** (larger NVMe), **`UUID=`** lines in `/etc/fstab`, survive reboot. |
| **`sdb` (1TB HDD)** | **Unpartitioned / unused** for a backup tier mount. | **ext4**, **`/backups`**, persistent **`fstab`** entry (UUID). |
| **High-churn paths** | Would have lived on RAID-backed **`/var`** if workloads ran there. | **Five bind mounts** from **`/platform/...`** → **`/var/lib/{rancher,kubelet,containerd,prometheus,loki}`** so churn targets the platform NVMe. |
| **Boot RAID (`sda`+`sdc`, md0–md2)** | Live OS root. | **Unchanged** — not reformatted; RAID still carries `/`, `/boot`, `/var`. |

So the node moved from “boot tier live, other disks reserved” to “**four-tier storage layout active**”: boot RAID + platform NVMe + database NVMe + backup HDD, with binds wiring k3s/observability paths to `/platform`.

### Tier mapping (implemented)

| Tier | Device | Approx. size | Mount | Label | Notes |
|------|--------|--------------|-------|-------|--------|
| Platform | `nvme1n1` | ~238.5G | `/platform` | `platform` | Smaller NVMe → platform churn (k3s/observability paths). |
| Database | `nvme0n1` | ~476.9G | `/data` | `alethos-data` | Larger NVMe → Postgres/Redis; not literal “1TB” — hardware is 512G-class (documented in BR B.4 / D). |
| Backup | `sdb` | ~931.5G | `/backups` | `backups` | **SATA HDD** — not `sdc` (RAID). ✓ |

### Filesystem layout

- **ext4** on all three tiers; whole-disk or single-partition pattern accepted per BR (B.3: `lsblk` shows ext4 on block device; UUIDs match `fstab`).
- **`/etc/fstab`:** Tier lines use **`UUID=`** with `defaults,nofail` (B.3, C.6). Base mounts listed **before** bind mounts ✓.

### UUIDs (from BR — confirm on node with `blkid` if needed)

| Mount | UUID | Label |
|-------|------|--------|
| `/platform` | `70a566e7-4834-48a3-8b5a-a97fc0383488` | `platform` |
| `/data` | `8650424b-1fff-4f16-80a8-43b8545f27ba` | `alethos-data` |
| `/backups` | `bded6290-6ef8-4c02-90a2-0fff87394ba0` | `backups` |

### Bind mounts (all five)

| Source | Target |
|--------|--------|
| `/platform/rancher` | `/var/lib/rancher` |
| `/platform/kubelet` | `/var/lib/kubelet` |
| `/platform/containerd` | `/var/lib/containerd` |
| `/platform/prometheus` | `/var/lib/prometheus` |
| `/platform/loki` | `/var/lib/loki` |

Options `bind,nofail` — consistent with task and failure-domain isolation (high-churn off boot RAID).

### Directories under `/data`

`postgres`, `postgres_wal`, `redis` — per task spec (BR C.5).

### Boot RAID

- **`sda` / `sdc` / `md0`–`md2`:** Reported **unchanged**; `mdstat` **`[2/2] [UU]`** (B.3, B.6).

### Validation claimed met (B.6)

- `findmnt` for `/platform`, `/data`, `/backups` and bind targets.
- `mount -a -v` succeeds.
- `lsblk -f` UUIDs match `fstab`.
- Cold reboot test passed (per BR narrative).

---

## 1. Architecture alignment

**Assessment: ✓ Pass**

| Criterion | Result | Notes |
|-----------|--------|--------|
| Platform tier off boot RAID | ✓ | `/platform` on dedicated NVMe; binds cover k3s/runtime paths. |
| Database tier isolated | ✓ | `/data` on separate NVMe; WAL/logical dirs prepared. |
| Backup tier on HDD (`sdb`) | ✓ | Correct failure domain; **not** on `sdc` (RAID). |
| `fstab` UUID-only for tier filesystems | ✓ | Matches task; boot lines may remain installer `by-id` (BR B.8). |
| Base mount before binds | ✓ | Order documented in C.6. |
| Boot tier not reformatted | ✓ | Only `nvme0n1`, `nvme1n1`, `sdb` touched for tiers. |
| TASK-0001 dependency respected | ✓ | RAID layout preserved. |

**Hardware naming vs docs:** Task text sometimes says “1TB database NVMe”; node has ~477G SN530. **Acceptable** — BR B.4 / Part D call this out; Documentation Agent should sync `project-context.json` and blueprints to **actual** sizes.

---

## 2. Security review

| Area | Summary |
|------|--------|
| **fstab** | UUID-based tier entries reduce boot-order swap risk; `nofail` avoids total boot hang if a tier is missing (understand tradeoff: service may start without data — acceptable for homelab with monitoring). |
| **Permissions** | `root:root`, `755` on tier roots per runbook; fine for pre-k3s; uids/gids for workloads adjusted in later tasks if needed. |
| **Secrets in repo** | UUIDs and labels appear in BR — normal for homelab; no passwords. |
| **Destructive ops** | Runbook stresses `lsblk` before `mkfs` — good. |

---

## 3. Infrastructure risk review

### Infra risk scoring (qualitative)

| Risk | Severity | Notes |
|------|----------|--------|
| Wrong-disk `mkfs` | **Critical** if occurs | Mitigated by explicit preconditions (C.0, C.1, Part D). |
| `sdb` vs `sdc` confusion | **High** if wrong | BR repeats: backup = **`sdb`**, **`sdc` = RAID** — aligned with `TASK_0002.md` “Live node truth”. |
| `nofail` + missing tier | Medium | Node may boot with incomplete storage; alert when observability lands. |
| Whole-disk vs `p1` drift | Low | Both valid; BR documents either pattern. |
| DB tier capacity vs “1TB” expectation | Low | Document actual ~477G to avoid capacity surprises. |

### Failure-domain analysis

- **Platform vs DB vs backup** remain separate mount trees — blast radius contained.
- **Binds** keep `/var/lib/*` hot paths off md-backed `/var` — aligns with Phase 0 lessons (do not mix OS disk with high churn).

### Rollback

- BR C.8 defines `fstab` restore + umount sequence + live USB recovery — **acceptable** for this task.

---

## 4. Observability check

| Item | Status |
|------|--------|
| Disk layout visibility | ✓ `lsblk -f`, `df -h`, `findmnt` in runbook. |
| RAID health | ✓ `cat /proc/mdstat` in checklist. |
| Metrics/alerts | Out of scope; capacity monitoring still recommended when Prometheus is deployed. |

---

## 5. Build report recommendations (non-blocking)

1. **Keep BR as canonical** — `BR-TASK-0002.md` is clear; minor duplication between “partition + p1” runbook (C.2–C.3) and “whole-disk ext4” live note (B.3) is **intentional** for different install paths.
2. **Retain `/etc/fstab.pre-task-0002`** on node until Planner closes task (BR B.9).
3. **Optional:** Add one line to Part B that `pass` numbers in `fstab` (e.g. `0 2` vs `0 1`) were validated for this Ubuntu release (if not already implicit).

---

## 6. Verification steps (operator spot-check)

Use these to **reconfirm** live state matches the build report (recommended before archiving the task).

```bash
lsblk -f
grep -v '^#' /etc/fstab | grep -v '^$'
findmnt -A | grep -E 'platform|/data|backups|rancher|kubelet|containerd|prometheus|loki'
df -h /platform /data /backups
cat /proc/mdstat
```

**Expected:**

- UUIDs for `/platform`, `/data`, `/backups` match `blkid` for the devices actually mounted.
- Five bind mounts present with sources under `/platform/...`.
- Capacities ~240G / ~477G / ~931G (within rounding).
- RAID arrays healthy, not degraded.

**Cold reboot:** Already claimed in BR; repeat if any `fstab` edit occurs after review.

---

## 7. Documentation updates (Documentation Agent)

Handled per **`BR-TASK-0002.md` Part E** and `tasks/TASK_0002.md`:

| File | Action |
|------|--------|
| `docs/system-blueprint.md` | Four-tier layout, devices, binds; optional UUIDs. |
| `docs/platform-overview.md` | Storage summary; **real** NVMe sizes. |
| `context/project-context.json` | `sdb` = backup, `sdc` = RAID; NVMe nodes; tiers implemented. |
| `context/Final_disk_layout_all_drives.md` | Align or add “alethos-node-01 live” note (per BR). |
| `tasks/todo.md` | Mark TASK-0002 **Complete** after doc sync. |
| `tasks/lessons.md` | Optional: rule “verify HDD identity before backup `mkfs`”. |

---

## 8. Recommendations

1. **Documentation Agent:** Replace any lingering “1TB NVMe” wording with **measured** size (~476.9G) or label “512G-class” to match hardware.
2. **Before TASK-0003:** Confirm ownership for `containerd`/`kubelet` paths if k3s expects specific uid/gid (may be no-op on fresh dirs).
3. **Disk space alerts:** When observability is deployed, alert on `/platform` and `/data` thresholds (80/90/95%).

---

## 9. Summary

- **Storage reality:** TASK-0002 **changed the live node** from “boot RAID only in use; NVMe + HDD reserved” to **persistent `/platform`, `/data`, `/backups`** plus **bind mounts** (subsection *Storage reality change* above). Boot RAID unchanged.
- **Correctness:** Tier mounts, UUID `fstab`, directory layout, and five bind mounts match `tasks/TASK_0002.md`. Backup on **`sdb`**, RAID on **`sda`/`sdc`** — consistent with live node truth.
- **Security:** Appropriate for Phase 0; no new exposure beyond standard disk UUIDs in docs.
- **Risks:** Well called out in BR Part D; wrong-disk risk mitigated by runbook discipline.
- **Observability:** Operational commands present; metrics deferred to later phases.
- **Docs:** Delegated to Documentation Agent per Part E.

**Verdict:** **PASS** — TASK-0002 implementation accepted per `BR-TASK-0002.md`; documentation and `tasks/todo.md` updates delegated to Documentation Agent / Planner.
