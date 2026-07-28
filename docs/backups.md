# Backup and recovery

**Status: PLANNED.** The backup tier is built and mounted. The automation, retention policy, and restore verification aren't. Until I've actually performed a restore, this node has a backup disk rather than backups.

---

## 1. Position

Single node, no HA, which makes recovery the whole resilience story:

- A failed boot SSD is survivable thanks to RAID1, though the failover boot test is still outstanding.
- A failed platform or database NVMe is **not** survivable by redundancy. Neither is mirrored.
- A bad deployment, a dropped table, or a corrupted database isn't a hardware problem at all, and RAID wouldn't help even if everything were mirrored.

For everything except a single boot-disk failure, the answer is **restore from backup**, which means the restore path has to be real and rehearsed rather than theoretical.

---

## 2. What gets backed up

| Target | Method | Frequency | Destination |
|--------|--------|-----------|-------------|
| PostgreSQL | `pg_dump` / `pg_basebackup` | Daily; WAL archived continuously once PITR lands | `/backups/postgres/` |
| Redis | RDB snapshot | Daily — cache and queue state, lower value | `/backups/redis/` |
| k3s cluster state | k3s snapshot (etcd/SQLite) | Daily and before any cluster change | `/backups/k3s/` |
| Kubernetes manifests | Git — this repo | Every commit | Remote repository |
| Application config and secrets | Encrypted export, sourced outside the repo | On change | `/backups/config/` |
| Host config (`/etc/fstab`, mdadm, k3s config) | Archived copy | On change | `/backups/host/` |

Nothing here gets backed up by copying live directories. PostgreSQL data files and Redis dumps are captured through their own tooling, because copying a live PV directory gives you a file set that looks like a backup and restores into a corrupt database.

---

## 3. Retention

Candidate policy, to be confirmed against actual data size once the workload is running:

| Class | Keep |
|-------|------|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

~931 GB of backup capacity against a ~477 GB database tier means retention depth is bounded by real data growth rather than by disk. I'll revisit the policy once Lemurjob has a measurable footprint.

---

## 4. RPO and RTO

Homelab-appropriate targets, written down so decisions have something to be measured against:

| Metric | Target | What it means |
|--------|--------|---------------|
| RPO | 24 h initially; ≤ 15 min once WAL archiving lands | Acceptable data loss |
| RTO | 4 h | Time to a working service after total node loss, assuming hardware is available |

The RTO number accounts for a real constraint: with a single node and no spare, "restore" includes acquiring or repairing hardware. The part I can actually measure is everything after that, meaning reinstall, remount tiers, restore data, redeploy.

---

## 5. Restore verification

This is the part that makes the rest of it count. A backup nobody has restored from is a guess.

| Drill | Cadence | Proves |
|-------|---------|--------|
| Restore Postgres dump into a scratch database and verify row counts | Monthly | The dump is valid and loadable |
| Restore k3s snapshot into a test context | Quarterly | Cluster state is recoverable |
| Full bare-metal rebuild rehearsal — reinstall, remount tiers, restore, redeploy | Annually, and after major changes | The whole procedure works and the runbook is accurate |
| Boot-disk failover test | Once, then after hardware changes | RAID1 actually fails over — **currently outstanding** |

Each drill produces a written record: what was run, how long it took, and what turned out to be wrong with the runbook. The runbook being wrong is the normal outcome of a first drill, and much better to find during a rehearsal.

---

## 6. Off-host replication

Everything above lives on `/backups`: a single HDD, in the same chassis, on the same power supply, in the same room as the data it protects. That covers disk failure and human error. It doesn't cover theft, fire, flood, or a PSU that takes the whole machine with it.

Off-host replication is a **required follow-up**, not an optional refinement. The candidates are object storage (S3 or equivalent) for encrypted archives, or `rsync` to a second physical machine. Encryption at rest is mandatory for anything leaving the node, since backups contain user data.

Until that exists, I'd rather state the limitation openly than leave it implied.

---

## 7. Monitoring the backups

Backups fail silently by default. Every item here becomes a required alert once the observability phase lands:

- **Last successful backup age** exceeds its schedule → high
- Backup job exit status non-zero → high
- `/backups` above 90 % → warning
- Backup file size deviates sharply from the recent baseline → warning (catches a "successful" run that produced an empty dump)
- Restore drill overdue → warning

---

## 8. Sequencing

| Step | Depends on |
|------|-----------|
| Automate Postgres dumps to `/backups` | Postgres deployed |
| Automate k3s snapshots | Cluster accepted |
| Retention and rotation | Backup automation running |
| Backup monitoring and alerts | Observability phase |
| First restore drill | One full retention cycle of real backups |
| WAL archiving / PITR | Restore drill passing |
| Off-host replication | Retention policy settled; encryption in place |
