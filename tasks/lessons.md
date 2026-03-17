# Lessons Repository
## Alethos Platform — Durable Engineering Knowledge

This file contains **durable engineering lessons** extracted from real task execution.

Its purpose is to:

- prevent repeated mistakes
- improve operator discipline
- enforce architectural consistency
- accelerate future task execution

---

# Usage Model

Lessons are used in two modes:

## 1. Prevention Mode (State 2)
Before executing a task:
- relevant lessons are surfaced
- warnings are injected
- precautions are applied

## 2. Learning Mode (State 7)
After completing a task:
- new lessons are extracted
- existing lessons may be refined
- rules are strengthened

---

# Index

## Categories

- [Infrastructure](#infrastructure)
- [Linux Operations](#linux-operations)
- [Architecture](#architecture)
- [Process / Workflow](#process--workflow)
- [Verification](#verification)
- [Security](#security)
- [Observability](#observability)

---

# Lesson Format

Each lesson follows this structure:

**[CATEGORY][ID] Title**

Context:
What situation produced this lesson?

Issue:
What failed, surprised, or required correction?

Root Cause:
Why did it happen?

Rule:
Reusable rule going forward.

Verification:
How to confirm the rule is being followed.

Tags:
[keywords for search]

---

# Infrastructure

## [INF-001] Always Wipe Disks Before RAID Setup

Context:
RAID setup encountered unexpected disk state during installation.

Issue:
Old metadata interfered with correct RAID initialization.

Root Cause:
Previous filesystem or RAID signatures were still present.

Rule:
Always wipe disks before configuring RAID.

Verification:
Run:

```bash
wipefs -a /dev/sdX
```

Then confirm:

```bash
lsblk -f
```

No residual filesystem or RAID signatures should appear.

Tags:
[raid, disks, setup, wipefs]


## [INF-002] Document and Use Consistent Device Mapping in Runbooks

Context:
TASK-0001 build and review referred to the second boot SSD; on the actual machine sda = Intel Pro 1500, sdc = Intel 520, sdb = 1TB HDD. Confusion between sdb and sdc could cause wrong-disk operations.

Issue:
Risk of wiping or partitioning the wrong disk (e.g. backup HDD) if device letters are assumed or used inconsistently.

Root Cause:
No single explicit "hardware mapping (this machine)" in the runbook; device names varied across commands and validation.

Rule:
In any runbook or build report that touches block devices, document the actual device mapping (e.g. "sda = DiskA, sdb = HDD, sdc = DiskB") at the top and use it consistently in all commands and validation.

Verification:
Grep the runbook for `/dev/sdX` and `/dev/nvme*`; confirm each appears in the mapping table and no device is used for two different roles.

Tags:
[runbook, devices, raid, homelab]

---

## [INF-003] Define In-Scope and Out-of-Scope Devices for Follow-on Storage Tasks

Context:
TASK-0002 will partition and mount NVMe and HDD for platform/backups; the second boot SSD (e.g. sdc) must not be touched.

Issue:
If a follow-on task describes "all non-boot disks" without naming them, an operator might include the second boot SSD and wipe the boot mirror.

Root Cause:
Follow-on task scope did not explicitly list which devices are in scope and which are out of scope.

Rule:
When a follow-on task partitions or formats disks, explicitly list devices in scope (e.g. nvme0n1, nvme1n1, sdb) and state out-of-scope devices (e.g. "Do not use sdc — second boot SSD").

Verification:
Task spec and build report for the follow-on task include "Devices in scope" and "Devices explicitly out of scope".

Tags:
[task-scope, storage, raid, follow-on]

---

## [INF-004] Verify Dual-Disk Boot Resilience by Failover Test

Context:
Dual ESP and GRUB on both SSDs were configured for RAID1 boot resilience.

Issue:
Without testing, it is unknown whether the system actually boots from the second disk if the first fails; review flagged "BIOS boot order / failover untested".

Root Cause:
Boot resilience was assumed from configuration rather than verified by failover.

Rule:
For dual-disk (or multi-disk) boot setups, perform a manual failover test: boot with one disk disabled or unplugged, confirm the system boots from the other, then re-enable and resync; document the outcome in the runbook or lessons.

Verification:
Runbook or lessons contain a "Failover test" step and its result (e.g. "Tested: boot from sdc with sda disconnected — OK").

Tags:
[boot, raid, resilience, uefi, grub]

---

## [INF-005] State Rollback Explicitly for Foundation/Install Tasks

Context:
Project rule: infrastructure tasks must include rollback procedure. TASK-0001 is an OS/RAID install with no in-place rollback.

Issue:
If rollback is not stated, operators may assume an in-place rollback exists; review recommended adding a short rollback note to the runbook.

Root Cause:
Rollback procedure was not written down for the foundation install task.

Rule:
For OS install or RAID foundation tasks, explicitly state in the runbook that rollback means reinstall from media (and restore from backup if applicable); there is no in-place rollback.

Verification:
Runbook contains a "Rollback" subsection stating the reinstall/restore approach.

Tags:
[rollback, runbook, foundation, install]

---


## [INF-006] Sync Secondary ESP After Kernel or GRUB Updates

Context:
Dual-ESP setup: primary ESP on first boot SSD, secondary ESP on second boot SSD, synced once via rsync after install.

Issue:
Kernel and GRUB updates typically rewrite only the primary ESP; the secondary ESP can become stale and fail to boot after an update.

Root Cause:
No process to refresh the secondary ESP when bootloader or kernel changes.

Rule:
After kernel or GRUB updates, sync the primary ESP to the secondary ESP (e.g. `rsync -a /boot/efi/ /mnt/efi2/` with secondary ESP mounted) so both boot disks remain current; consider a post-update hook or documented manual step.

Verification:
After a kernel or `grub-install` update, confirm the secondary ESP contains the same EFI/ubuntu (or equivalent) files as the primary; document or automate the sync step.

Tags:
[efi, grub, boot, raid, dual-esp]

---

# Linux Operations

## [LINUX-001] Verify Mount Points After Configuration

Context:
Mount configuration completed but system behavior inconsistent.

Issue:
Mount points were not correctly applied or persisted.

Root Cause:
Incorrect fstab entry or missing mount verification.

Rule:
Always verify mounts immediately after configuration.

Verification:
Run:

```bash
mount | grep <mountpoint>
df -h
```

Ensure expected device is mounted at correct location.

Tags:
[mount, filesystem, fstab]

---

## [LINUX-002] Plan Post-Boot Network Fallback for Headless Installs

Context:
Ubuntu Server minimal install on the platform node; installer network autoconfiguration failed during installation.

Issue:
Headless install could stall or leave the system without network if the operator expects DHCP to work during the installer.

Root Cause:
Installer-time DHCP/network is not always reliable on all hardware or networks.

Rule:
For headless or minimal OS installs, plan a post-boot network configuration fallback (e.g. Netplan, manual edit of `/etc/netplan/*.yaml`) so the system can be configured after first boot if installer-time network fails.

Verification:
Runbook includes an "If installer has no network" or "Post-boot network" section with concrete steps (e.g. `netplan apply`, `ip addr`, `ping`).

Tags:
[install, network, netplan, headless]

---


# Architecture

## [ARCH-001] Enforce Storage Tier Separation

Context:
Platform design includes multiple storage tiers.

Issue:
Workloads risk being colocated incorrectly.

Root Cause:
Lack of strict enforcement of storage boundaries.

Rule:
Always maintain separation between:
- OS
- platform runtime
- database
- backups

Verification:
Run:

```bash
lsblk
```

Ensure each tier maps to the correct device.

Tags:
[storage, architecture, separation]

---

# Process / Workflow

## [PROC-001] Never Skip Verification Phase

Context:
Implementation was assumed correct without validation.

Issue:
Hidden issues surfaced later.

Root Cause:
Verification step was skipped or incomplete.

Rule:
Every task must pass explicit verification before acceptance.

Verification:
Reviewer must produce:
- verification checklist
- PASS / FAIL status

Tags:
[workflow, verification]

---

# Verification

## [VER-001] Always Validate System State Using Commands

Context:
System assumed correct without inspecting actual state.

Issue:
Mismatch between expected and actual system configuration.

Root Cause:
No direct system inspection performed.

Rule:
Always validate using system commands, not assumptions.

Verification:
Examples: `lsblk`, `df -h`, `cat /proc/mdstat`

Tags:
[verification, inspection]

---

# Security

## [SEC-001] Avoid Implicit Trust in Default Configurations

Context:
Default system settings assumed secure.

Issue:
Defaults may expose system or reduce resilience.

Root Cause:
No explicit validation of security posture.

Rule:
Always review and validate security-relevant defaults.

Verification:
Check: permissions, open ports, services running.

Tags:
[security, defaults]

---

# Observability

## [OBS-001] Ensure Observability Exists Before Scaling

Context:
System scaled without monitoring in place.

Issue:
No visibility into system behavior under load.

Root Cause:
Observability added too late.

Rule:
Observability must be in place before scaling or production use.

Verification:
Confirm: logs accessible, metrics available, dashboards exist.

Tags:
[observability, monitoring]

---

# Guidelines for Adding Lessons

When adding a new lesson:

- Ensure it is reusable
- Ensure it is based on real experience
- Keep it concise and actionable
- Assign a category and ID
- Add verification steps
- Add tags for searchability

**Group lessons by phase** (infra, API, deployment) whenever possible.

## Anti-Patterns

Do not add:

- vague statements
- one-time observations
- redundant lessons
- unverified assumptions

---

# Purpose Reminder

This file is the memory system of engineering discipline. It ensures that:

- mistakes are not repeated
- knowledge compounds over time
- execution becomes faster and safer
