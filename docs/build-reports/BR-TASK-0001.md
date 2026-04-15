TASK-0001 — Ubuntu Server installation + RAID1 boot mirror
==========================================================

Goal
----

Install Ubuntu Server as a headless, SSH-accessible OS on the Lenovo P510, using the two Intel 180GB SATA SSDs as an mdadm RAID1 boot tier in UEFI mode, with the following layout:

- Per disk (`sda`, `sdc`): 512MiB EFI, 2GiB BOOT RAID member, 50GiB ROOT RAID member, remainder VAR RAID member. **Hardware mapping (this machine):** sda = Intel Pro 1500, sdc = Intel 520, sdb = 1TB HDD (backup tier only; not part of boot).
- RAID1 arrays: `/dev/md0` → `/boot`, `/dev/md1` → `/`, `/dev/md2` → `/var`.
- Dual EFI partitions: both Intel SSDs contain valid EFI bootloaders, and GRUB is installed to both disks.

This provides a resilient, predictable OS foundation that keeps the boot tier separate from high-churn workloads and from database/storage tiers used in later tasks.

Context and Configuration Details
---------------------------------

Phase / roadmap alignment:

- Phase: Phase 0 — Platform Foundation (Phase 0A/0B: hardware + OS baseline and boot/storage architecture).
- Active tasks: TASK-0001 (this task), with TASK-0002 (storage tier mounts) queued next.
- Design principles drawn from `context/project-context.json`, `context/Homelab_Hardware_setup.json`, and `context/Final_disk_layout_all_drives.md`:
  - OS and boot on a dedicated RAID1 SATA SSD tier.
  - Platform workloads and observability on `/platform` (NVMe).
  - Databases on `/data` (other NVMe).
  - Backups on `/backups` (HDD).
  - `/var` separated from `/` to limit blast radius from logs and system churn.

Hardware involved:

- Boot tier:
  - `INTEL_SSDSC2BF180AL` (Intel SSD Pro 1500, ~180GB) → `/dev/sda`.
  - `INTEL_SSDSC2CW180A3` (Intel SSD 520, ~180GB) → `/dev/sdc`.
- Other storage tiers (left untouched for this task, reserved for TASK-0002):
  - `/dev/nvme0n1` (~480–500GB NVMe, actual reported size ~476.9G) → future `/data` or `/platform` per final mapping.
  - `/dev/nvme1n1` (~240GB NVMe, actual reported size ~238.5G) → future `/platform` or `/data` per final mapping.
  - `/dev/sdb` (~1TB HDD, actual reported size ~931.5G) → future `/backups`.

Partition and RAID configuration implemented:

- Both Intel SSDs (`/dev/sda` and `/dev/sdc`) use GPT partition tables with identical geometry:
  - Partition 1: 512MiB, type “EFI System Partition” (ESP), filesystem FAT32, intended mount `/boot/efi`.
  - Partition 2: 2GiB, raw (used as mdadm RAID1 member for `/boot`).
  - Partition 3: 50GiB, raw (used as mdadm RAID1 member for `/`).
  - Partition 4: remaining space, raw (used as mdadm RAID1 member for `/var`).
- RAID arrays (mdadm software RAID1):
  - `/dev/md0` — RAID1 over `/dev/sda2` and `/dev/sdc2` → mounted as `/boot` (ext4).
  - `/dev/md1` — RAID1 over `/dev/sda3` and `/dev/sdc3` → mounted as `/` (ext4).
  - `/dev/md2` — RAID1 over `/dev/sda4` and `/dev/sdc4` → mounted as `/var` (ext4).

Boot / EFI configuration:

- UEFI-only boot mode configured in firmware (no Legacy/CSM).
- Primary EFI partition:
  - `/dev/sda1` formatted as FAT32 and mounted at `/boot/efi`.
  - GRUB installed to `/dev/sda` and associated with the ESP on `/dev/sda1`.
- Secondary EFI partition:
  - `/dev/sdc1` formatted as FAT32 and used as a second ESP.
  - Contents rsynced from `/boot/efi` to ensure a duplicate, bootable EFI environment on the second SSD.
- GRUB installed onto the MBR/EFI targets of **both** SSDs:
  - `/dev/sda` and `/dev/sdc` both have GRUB, so the system can boot from either disk if one fails.

Networking / access:

- Ubuntu Server LTS (minimal install) deployed, headless-only (no GUI).
- Initial installer network autoconfiguration failed, so:
  - Install proceeded without installer-time network.
  - After first boot, Netplan was used to configure the primary NIC (e.g. `eno1`) with DHCP (`dhcp4: true`).
  - Outbound networking validated via `ping`.
- OpenSSH server installed during the OS installation, and SSH login confirmed after network configuration.

Decisions Made
--------------

- **Boot on RAID1 over SATA SSDs, not NVMe**:
  - Aligns with project context that NVMe is reserved for high-churn platform and data tiers.
  - Reduces risk of firmware quirks around NVMe boot on this hardware.

- **Three separate RAID1 arrays (`/boot`, `/`, `/var`) instead of a single large array**:
  - `/var` is kept separate from `/` to contain failures from logs and system churn.
  - `/boot` isolation ensures a simple, resilient boot environment.

- **Dual ESP approach (ESP on both SSDs) instead of single ESP + manual sync**:
  - Each SSD has its own EFI partition, and both contain bootloader files.
  - This allows clean fail-over: if one SSD fails, firmware can still boot from the other.

- **Installer used only for the two Intel SSDs; NVMe and HDD left untouched**:
  - Ensures strict adherence to tier separation and keeps TASK-0001 bounded to the boot tier only.

Commands Executed
-----------------

This section lists the representative commands the Operator is expected to have run, in approximate order. On aluna-node-01 the boot SSDs are `sda` and `sdc`; `sdb` is the HDD. NIC name may vary.

Installer environment (before partitioning):

```bash
# Identify disks
lsblk -o NAME,SIZE,MODEL,TYPE

# Wipe old partition tables and RAID metadata on Intel SSDs
sudo wipefs -a /dev/sda
sudo wipefs -a /dev/sdc
sudo sgdisk --zap-all /dev/sda
sudo sgdisk --zap-all /dev/sdc

# Create GPT partitions on both disks (run in a loop)
for disk in /dev/sda /dev/sdc; do
  sudo parted --script "$disk" \
    mklabel gpt \
    mkpart ESP fat32 1MiB 513MiB \
    set 1 esp on \
    mkpart boot 513MiB 2561MiB \
    mkpart root 2561MiB 55201MiB \
    mkpart var 55201MiB 100%;
done

# Create RAID1 arrays over the corresponding partitions
sudo mdadm --create /dev/md0 --level=1 --metadata=1.2 --raid-devices=2 /dev/sda2 /dev/sdc2
sudo mdadm --create /dev/md1 --level=1 --metadata=1.2 --raid-devices=2 /dev/sda3 /dev/sdc3
sudo mdadm --create /dev/md2 --level=1 --metadata=1.2 --raid-devices=2 /dev/sda4 /dev/sdc4
- Note: the above `nvme0n1`/`nvme1n1` role mapping reflects current detected sizes; final role assignment (which NVMe is platform vs. database tier) should be confirmed in TASK-0002 based on `context/Homelab_Hardware_setup.json` and actual device enumeration.


# Verify arrays in installer shell
cat /proc/mdstat
```

Ubuntu installation (guided via installer UI):

- Custom storage layout:
  - `/dev/md0` → ext4 → `/boot`.
  - `/dev/md1` → ext4 → `/`.
  - `/dev/md2` → ext4 → `/var`.
  - `/dev/sda1` → EFI System Partition (FAT32) → `/boot/efi`.
  - `/dev/sdc1` (second ESP) left unmounted during install.
- Ubuntu Server minimal install with OpenSSH server enabled, admin user with sudo.

Post-install (on the running system):

```bash
# Verify block devices, filesystems, and mounts
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
df -h
cat /proc/mdstat

# If needed: format and mount primary EFI as FAT32 ESP
sudo mkfs.vfat -F 32 -n EFI /dev/sda1
sudo parted /dev/sda set 1 esp on
sudo mkdir -p /boot/efi
echo '/dev/sda1 /boot/efi vfat umask=0077 0 1' | sudo tee -a /etc/fstab
sudo mount /boot/efi

# Persist RAID configuration and ensure arrays assemble at boot
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# Install GRUB on both SSDs (sda and sdc; sdb is the HDD, do not use for boot)
sudo grub-install /dev/sda
sudo grub-install /dev/sdc
sudo update-grub

# Prepare and sync secondary EFI partition (sdc1)
sudo mkfs.vfat -F 32 -n EFI2 /dev/sdc1
sudo mkdir -p /mnt/efi2
sudo mount /dev/sdc1 /mnt/efi2
sudo rsync -a /boot/efi/ /mnt/efi2/
sudo umount /mnt/efi2

# Networking via Netplan (example for eno1)
sudo nano /etc/netplan/*.yaml   # set eno1: dhcp4: true
sudo netplan apply
ip addr
ping -c 2 8.8.8.8

# Base packages and SMART
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git htop nvme-cli smartmontools
sudo systemctl enable smartd
sudo systemctl start smartd
sudo smartctl -H /dev/sda
sudo smartctl -H /dev/sdc
```

Validation
----------

Boot and RAID layout:

- `cat /proc/mdstat` shows three active RAID1 arrays:
  - `md0` with members `/dev/sda2` and `/dev/sdc2`, state `clean` or `active`.
  - `md1` with members `/dev/sda3` and `/dev/sdc3`, state `clean` or `active`.
  - `md2` with members `/dev/sda4` and `/dev/sdc4`, state `clean` or `active`.
- `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT` confirms:
  - `/dev/md0` (ext4) mounted at `/boot`.
  - `/dev/md1` (ext4) mounted at `/`.
  - `/dev/md2` (ext4) mounted at `/var`.
  - `/dev/sda1` (vfat) mounted at `/boot/efi`.
  - `/dev/sdc1` (vfat) present as a second ESP (not necessarily mounted persistently).

Filesystem usage:

- `df -h` shows:
  - Sufficient free space on `/` (from `/dev/md1`) for OS and base services.
  - Sufficient free space on `/var` (from `/dev/md2`), which will later offload container/observability runtime to `/platform`.

Bootloader and EFI:

- `sudo efibootmgr -v` lists at least one Ubuntu boot entry pointing at EFI files on an Intel SSD.
- Both `/boot/efi` (on `/dev/sda1`) and the secondary ESP (`/dev/sdc1` when mounted) contain `/EFI/ubuntu/` with GRUB/shim files.
- A reboot test (with no installer media) succeeds and returns to the Ubuntu login prompt.

Networking:

- `ip addr` shows the primary NIC (e.g. `eno1`) with an IPv4 address via DHCP.
- `ping -c 2 8.8.8.8` (and/or to the local gateway) succeeds with no packet loss.
- SSH connection from the development machine to the server works using the admin account.

Health:

- `sudo smartctl -H /dev/sda` and `sudo smartctl -H /dev/sdc` both report a passing overall-health status.

Risks and Open Questions
------------------------

- **Mixed-model SSD RAID**:
  - The RAID1 mirror combines an Intel SSD Pro 1500 and an Intel SSD 520. Capacity is constrained to the smaller effective size, and wear characteristics may differ.
  - Risk is acceptable in a homelab context but should be monitored via SMART and RAID health checks.

- **BIOS/firmware boot behavior**:
  - While both SSDs are prepared with ESPs and GRUB, actual fail-over behavior depends on firmware boot order and how the BIOS chooses alternative boot entries upon disk failure.
  - A manual test (temporarily disabling one disk in BIOS or unplugging it) is recommended to confirm that the system can boot from the surviving SSD in a degraded RAID state.

- **Installer variations**:
  - Different Ubuntu LTS versions may present slightly different partition/RAID UI flows, so the exact sequence may need adaptation if the installer UX changes in future installs.

- **EFI sync process**:
  - Currently, EFI partitions are synced via a one-time `rsync`. Long-term, a periodic or event-driven sync mechanism may be desirable if bootloader updates occur (e.g. kernel upgrades), though modern updates typically rewrite the primary ESP only.

Docs and Context to Update
--------------------------

These updates are for the Documentation Agent / Context Curator and Planner; they are **not** performed by this build step but are required to keep the repo as the single source of truth:

- `docs/system-blueprint.md`:
  - Document the concrete boot tier implementation:
    - UEFI-only boot on two Intel 180GB SSDs.
    - mdadm RAID1 arrays `/dev/md0` → `/boot`, `/dev/md1` → `/`, `/dev/md2` → `/var`.
    - Dual ESP strategy and GRUB on both SSDs.
  - Confirm that NVMe (platform, database) and HDD (backups) remain unpartitioned at this stage.

- `docs/platform-overview.md`:
  - Update the platform storage and boot overview to reference the implemented layout.
  - Highlight separation of OS, platform, database, and backup tiers and how this supports the “mini production cluster” model.

- `context/project-context.json`:
  - Under `current_storage_layout` and `boot_raid1_partition_plan`, mark the boot tier as “implemented” rather than purely “intent”.
  - Optionally add a timestamped note under `meta.change_log` describing that the RAID1 boot mirror and partition scheme are now live.

- `tasks/todo.md`:
  - After Reviewer-Verifier approval and Documentation Agent updates, Planner should eventually mark TASK-0001 as `Status: Complete`.

- `tasks/lessons.md`:
  - If any surprises or corrections occurred during the actual install (e.g., EFI UX limitations in the installer, Netplan/dhcp quirks), the Lessons Agent should record them as new durable rules for future OS/RAID tasks.

Assumptions
-----------

- `sda` and `sdc` correspond to the two Intel 180GB SSDs as described in the hardware context; `sdb` is the 1TB HDD (backup tier only).
- The P510 runs in UEFI mode with SATA in AHCI, and the firmware can boot from either Intel SSD’s ESP.
- Ubuntu Server LTS minimal installer is used, with OpenSSH enabled during installation.
- NVMe and HDD devices are reserved exclusively for platform, database, and backup tiers and are not used by TASK-0001.
- Network connectivity (via DHCP or static configuration) is available once Netplan is configured, enabling SSH access and package updates.

Follow-up Tasks
---------------

- **TASK-0002 — Storage tier mount configuration**:
  - Partition and format **only** `/dev/nvme0n1`, `/dev/nvme1n1`, and `/dev/sdb` (HDD). Do **not** use `/dev/sdc` — it is the second boot SSD.
  - Mount them as `/platform`, `/data`, and `/backups` respectively.
  - Create and bind-mount runtime directories:
    - `/platform/rancher` → `/var/lib/rancher`
    - `/platform/containerd` → `/var/lib/containerd`
    - `/platform/kubelet` → `/var/lib/kubelet`
    - `/platform/prometheus` → `/var/lib/prometheus`
    - `/platform/loki` → `/var/lib/loki`
  - Ensure all mounts persist via `/etc/fstab` using UUIDs.

- **Verification / review pass**:
  - Reviewer-Verifier Agent should confirm RAID layout, boot resilience, and adherence to homelab storage principles.
  - Any additional risks or necessary checks should be captured and fed into `tasks/lessons.md` and the blueprint.

- **Longer-term**:
  - Decide on filesystem strategy (ext4 vs ZFS) for platform, database, and backup tiers as described in context.
  - Introduce backup automation and restore drills once database and workloads are present (later phases).

