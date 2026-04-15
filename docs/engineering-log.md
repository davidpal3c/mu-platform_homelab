# Engineering Log

Chronological record of significant platform changes for traceability and context reload.

Format: **Date | Task ID | Summary of change | Architectural impact**

---

2026-03-15 | TASK-0001 | Ubuntu Server installation and RAID1 boot configuration completed on aluna-node-01. Boot tier: two Intel SATA SSDs (sda + sdc) in mdadm RAID1 (md0→/boot, md1→/, md2→/var), dual ESP, GRUB on both disks. NVMe and HDD (sdb) reserved for TASK-0002. | Boot tier implemented and isolated from platform/DB/backup tiers; storage tier separation established; foundation for Phase 0 storage mount task.

2026-03-15 | TASK-0002 | Storage tiers mounted on aluna-node-01: /platform on nvme1n1 (~238G), /data on nvme0n1 (~477G), /backups on sdb (~931G); ext4; UUID-only fstab lines with defaults,nofail; five bind mounts from /platform to /var/lib/{rancher,kubelet,containerd,prometheus,loki}; post-reboot validation per BR-TASK-0002. | Platform, database, and backup failure domains are live; high-churn paths off boot RAID; k3s-ready bind layout; doc sizes corrected from nominal 1TB DB to 512G-class hardware.

2026-03-27 | TASK-0001 | Review/verification closure notes: `docs/build-reports/reviews/REVIEW-TASK-0001.md` **PASS** on aluna-node-01 (`lsblk` confirmed md0/md1/md2, dual ESP, sda+sdc RAID, sdb HDD untouched). **Corrections applied in narrative:** second boot SSD is **`sdc`** (Intel 520), not `sdb`; backup/HDD is **`sdb`**. Build report follow-up text aligned so TASK-0002 scope never targets `sdc`. **Open follow-ups (lessons):** manual disk failover boot test still recommended (INF-004); sync secondary ESP after kernel/GRUB updates (INF-006). | Runbooks and downstream tasks match live device mapping; residual risk is unexercised failover until tested.

2026-03-27 | TASK-0002 | Review/verification closure notes: `docs/build-reports/reviews/REVIEW-TASK-0002.md` **PASS**; implementation matches `tasks/TASK_0002.md` and `BR-TASK-0002` (tier UUIDs, base-before-bind `fstab`, RAID `[UU]` unchanged). **Corrections / doc hygiene:** legacy assumption “backup on sdc1” rejected — on this node **`sdc` is RAID**; backup tier is **`sdb`**. Nominal “1TB” DB NVMe wording to be replaced in blueprints/context with measured ~476.9G where still present (see lesson PROC-002). **Verification spot-check (operator):** `lsblk -f`, `grep -v '^#' /etc/fstab`, `findmnt`, `df -h` on tier mounts, `cat /proc/mdstat`. | Four-tier storage truth frozen for TASK-0003 (k3s); wrong-disk risk mitigated by explicit in/out-of-scope device lists in task + BR.

2026-03-27 | Meta | **Lessons Agent / durable rules:** `tasks/lessons.md` updated with Phase 0 Infrastructure grouping; INF-002–007, LINUX-002–004, PROC-002, VER-002 from TASK-0001/0002 outcomes and reviews. Operator confirmation workflow per `context/AGENTS.md` (Lessons Agent). | Engineering discipline and repo-as-memory for storage, `fstab`, and device enumeration; see `tasks/lessons.md` index and tags for retrieval.

2026-03-29 | Meta | **Doc layout:** Split **platform umbrella** (`docs/aluna-platform/` — ontology, platform Mermaid diagrams, `source/`) from **Aluna Context workload** (`docs/aluna-context/` — preserved `architecture-diagrams/` + PNGs). Removed duplicate ontology/diagrams from Context folder; cross-links updated across README and blueprint. | Clear separation: Aluna vs single-product Context docs.

2026-03-29 | Meta | **Aluna branding:** Added platform ontology and numbered Mermaid diagrams (now under `docs/aluna-platform/`), updated root `README.md`, homelab docs, tasks, build reports, and `context/project-context.json` (hostname **aluna-node-01**, keys `device_mapping_aluna_node_01` / `block_device_aluna_node_01`). Ext4 volume label **`alethos-data`** on `/data` retained as as-built / `blkid` truth. Runtime diagram labels updated to **Aluna Context API**. | Single vocabulary; git remote may remain `alethos_platform` until renamed.

2026-03-28 | TASK-0002 / Docs | **Documentation Agent pass:** `docs/system-blueprint.md` and `docs/platform-overview.md` headers updated to cite `REVIEW-TASK-0002.md` **PASS** and the “storage reality change” (persistent tiers + binds vs boot-only). `context/Homelab_Hardware_setup.json` aligned to as-built **aluna-node-01** (sda+sdc RAID, `nvme1n1`/`nvme0n1` platform/DB, `sdb` backup); `tasks/todo.md` and `tasks/TASK_0002.md` NVMe role text corrected. `context/project-context.json` `meta.change_log` entry added. | Repo context no longer contradicts live mapping; nominal “1TB DB NVMe” / wrong `sdb`/`sdc` roles removed from hardware JSON and task tables.
