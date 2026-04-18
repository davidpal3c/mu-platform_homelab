# Backlog — Aluna Platform Homelab

Ideas and follow-ups **not** yet promoted to `tasks/todo.md`. Phase-appropriate only; no API feature work before roadmap gates.

---

## Storage / disk identity (requires dedicated migration task)

- **Optional ext4 volume relabel:** `alethos-data` → `aluna-data` on `/data` using `tune2fs -L` (or equivalent) **only** with:
  - approved **migration + rollback** task spec
  - `blkid` / `fstab` UUID impact analysis (UUID unchanged; label-only vs any tooling that keys on label)
  - downtime / remount plan
  - **do not** rewrite completed TASK-0002 build/review artifacts for branding alone

- **ZFS vs ext4** on future nodes or greenfield reinstall (see `context/project-context.json` `filesystem_strategy`)

- **Prometheus/Loki retention and quotas** on `/platform` once TASK-0004 lands

---

## Platform / k8s

- **Ingress implementation choice** document (Traefik vs nginx) if not fixed during TASK-0005
- **cert-manager** + DNS-01 vs HTTP-01 strategy for homelab domain
- **k3s etcd snapshot** schedule and restore drill (operational maturity — later phase)

---

## Naming / docs hygiene

- **Git remote** name vs `alethos_platform` — document-only unless operator renames remote
- **Diagram PNG regeneration** when namespace names in Mermaid sources fully align with TASK-0003 baseline (`ceiba`, `aluna-context`, …) (coordinate with TASK-0006)

---

## Security / hardening (post baseline)

- SSH keys-only, `ufw`/nftables baseline, fail2ban optional
- **SEC-001** follow-up: explicit port inventory after k3s + ingress

---

## Aluna products (deferred by roadmap)

- Ceiba / Aluna Mira / Aluna Terra doc trees under `docs/` when workloads exist
- Aluna Context API deploy, Celery, AI enrichment — **explicitly later phases**
