---
Category: Procedure
Command: ssh, vi, chown, ls, lsdasd, dasd_configure, dasdfmt, fdasd, pvcreate, vgs, vgextend, vgcreate, lvcreate, lvs, mkreiserfs, cp
Date: 2011-07-14
DocType: How-to guide
Environment: Production
Filename: fstab
Filesystem: /etc, /etc/fstab, /dev, /tmp, /intranet27
OS: Linux
Product: SUSE Linux Enterprise Server 11, s390x, Domino, ReiserFS
Scope: Infrastructure
Server: lxzlin01, lxzlin02
Subsystem: LVM
Tag: dasd
Topic: Installation, Configuration
---

# IBM zLinux - DASD Provisioning and LVM Storage Setup for a New Domino Instance
 
## Overview
 
This procedure documents the full storage provisioning workflow performed on an IBM z/Architecture
Linux guest (`lxzlin02`, SUSE Linux Enterprise Server 11 on s390x) to bring a new Lotus Domino
mail instance (`intranet27`) into service.
 
The workflow covers:
 
- Correcting SSH client configuration to reach the target host
- Bringing DASD devices online from the hypervisor channel subsystem
- Low-level DASD formatting (`dasdfmt`) and partition creation (`fdasd`)
- LVM stack setup: Physical Volumes, Volume Groups, and Logical Volumes
- ReiserFS filesystem creation on each logical volume
The storage layout mirrors the existing `intranet25` and `intranet26` instances:
one **data** volume, one **translog** volume, and one **mail** volume (spanning three DASDs).
 
> **WARNING:** `dasdfmt` is a destructive operation. All data on the target DASD is erased.
> Confirm device Bus-IDs carefully before proceeding.
 
---
 
## Background - What Is DASD?
 
**DASD** (Direct Access Storage Device) is the IBM term for a disk storage unit attached
to a mainframe or IBM Z system. Unlike conventional block devices (e.g. SCSI or SATA disks)
that communicate over a PCI or SAS bus, DASDs are connected through IBM's **channel subsystem**
- a dedicated I/O fabric that offloads data transfer from the CPU entirely.
 
Each DASD is identified by a **Bus-ID** in the format `<css>.<ssid>.<devno>` (e.g. `0.0.0317`),
where `css` is the channel subsystem, `ssid` is the subchannel set, and `devno` is the
four-hex-digit device number allocated by the system administrator.
 
The most common DASD type in IBM Z environments is **ECKD** (Extended Count Key Data),
a track-and-cylinder geometry format that requires a low-level format (`dasdfmt`) before
the device can be used. This is fundamentally different from conventional disks, where the
physical format is done at the factory. ECKD geometry is expressed in cylinders × heads
(tracks per cylinder), and the block size must be explicitly chosen - `4096` bytes is
standard for modern Linux workloads.
 
The other type visible in this environment is **FBA** (Fixed Block Architecture), which
presents a flat sector-addressable surface similar to a conventional disk. FBA devices do
not require `dasdfmt`.
 
Under Linux on IBM Z (zLinux), the kernel exposes each active DASD as a block device
under `/dev/dasdX` (where `X` is an alphabetic suffix assigned in discovery order).
A DASD must be explicitly brought **online** via `dasd_configure` before the kernel
recognises it - devices allocated to the LPAR but not yet activated appear as `offline`
in `lsdasd -a` output.
 
The toolchain used in this procedure maps directly to the DASD lifecycle:
 
| Tool | Role |
|---|---|
| `lsdasd` | List all DASDs known to the LPAR and their online/offline state |
| `dasd_configure` | Set a DASD online or offline by Bus-ID |
| `dasdfmt` | Low-level format: writes track headers and sets block size |
| `fdasd` | Partition a DASD by writing a VTOC (Volume Table of Contents) |
 
Once partitioned, a DASD partition (e.g. `/dev/dasdv1`) is a standard Linux block device
and can be used directly with LVM (`pvcreate`, `vgcreate`, etc.) like any other disk.
 
---
 
## Prerequisites
 
- Root access (or `sudo`) on `lxzlin02`.
- DASD Bus-IDs to provision have been allocated and verified with the storage/mainframe team.
- Target mount points exist or will be created under `/intranet27/`.
- The operator workstation (`lxzlin01`) must have a valid SSH client config entry for `lxzlin02`.
---
 
## Procedure
 
### Step 1 - Fix SSH Client Configuration
 
The first connection attempt from `lxzlin01` to `lxzlin02` failed with
`Could not resolve hostname lxzlin02: Name or service unknown`. The `~/.ssh/config` file
was owned by `root` instead of the connecting user, preventing writes.
 
```bash
# On lxzlin01 - fix ownership of the SSH config file
sudo chown jdoe /home/jdoe/.ssh/config
```
 
Then add the `lxzlin02` host alias block to `~/.ssh/config` (hostname/IP mapping) and
re-attempt the connection:
 
```bash
ssh lxzlin02
```
 
Accept the RSA host key fingerprint on first connection. Subsequent connections will
use the cached key from `~/.known_hosts`.
 
---
 
### Step 2 - Identify Offline DASD Devices
 
Log in to `lxzlin02` as root and list all offline DASD devices assigned to this LPAR:
 
```bash
lsdasd -a | grep offline
```
 
Devices confirmed offline and ready for provisioning in this session:
 
| Bus-ID    | Purpose                        |
|-----------|--------------------------------|
| 0.0.0316  | `softdomvg` extension (2.3 GB) |
| 0.0.0317  | `intranet27datavg` (23 GB)     |
| 0.0.0321  | `intranet27translogvg` (7 GB)  |
| 0.0.0318  | `intranet27mailvg` member 1 (23 GB) |
| 0.0.0319  | `intranet27mailvg` member 2 (23 GB) |
| 0.0.0320  | `intranet27mailvg` member 3 (23 GB) |
 
---
 
### Step 3 - Bring DASDs Online
 
Use `dasd_configure` to set each device online. Repeat for every Bus-ID listed above.
 
```bash
dasd_configure 0.0.0316 1 0
dasd_configure 0.0.0317 1 0
dasd_configure 0.0.0321 1 0
dasd_configure 0.0.0318 1 0
dasd_configure 0.0.0319 1 0
dasd_configure 0.0.0320 1 0
```
 
The arguments are: `<bus-id> <online=1> <ro=0>`. Verify each device appears active:
 
```bash
lsdasd
```
 
Each device will be assigned a kernel block device name (`dasdu`, `dasdv`, `dasdw`, …).
 
---
 
### Step 4 - Low-Level Format Each DASD
 
DASD devices on z/Architecture require a low-level format before use. Use `dasdfmt` with
a 4096-byte block size and compatible disk layout (`-p`). The command prints geometry
and prompts for confirmation.
 
```bash
# Format each device - replace /dev/dasdX with the actual device name
dasdfmt -b 4096 -f /dev/dasdu -p    # 0.0.0316 - softdomvg extension
dasdfmt -b 4096 -f /dev/dasdv -p    # 0.0.0317 - intranet27 data
dasdfmt -b 4096 -f /dev/dasdw -p    # 0.0.0321 - intranet27 translog
dasdfmt -b 4096 -f /dev/dasdx -p    # 0.0.0318 - intranet27 mail (1/3)
dasdfmt -b 4096 -f /dev/dasdy -p    # 0.0.0319 - intranet27 mail (2/3)
dasdfmt -b 4096 -f /dev/dasdz -p    # 0.0.0320 - intranet27 mail (3/3)
```
 
Type `yes` at the confirmation prompt for each device. Formatting a 23 GB ECKD DASD
(32 759 cylinders) takes several minutes.
 
---
 
### Step 5 - Create DASD Partitions
 
`fdasd -a` auto-creates a single partition spanning the entire DASD and writes the
VTOC (Volume Table of Contents), which is the IBM equivalent of a partition table.
 
```bash
fdasd -a /dev/dasdu
fdasd -a /dev/dasdv
fdasd -a /dev/dasdw
fdasd -a /dev/dasdx
fdasd -a /dev/dasdy
fdasd -a /dev/dasdz
```
 
This produces partition devices: `/dev/dasdu1`, `/dev/dasdv1`, `/dev/dasdw1`,
`/dev/dasdx1`, `/dev/dasdy1`, `/dev/dasdz1`.
 
---
 
### Step 6 - Create Physical Volumes
 
Initialize each partition as an LVM Physical Volume:
 
```bash
pvcreate /dev/dasdu1
pvcreate /dev/dasdv1
pvcreate /dev/dasdw1
pvcreate /dev/dasdx1
pvcreate /dev/dasdy1
pvcreate /dev/dasdz1
```
 
> **NOTE:** Always use the partition device (`dasdX1`), not the raw device (`dasdX`).
> LVM on DASD requires the VTOC partition, not the whole disk.
 
---
 
### Step 7 - Extend Existing Volume Group (`softdomvg`)
 
The DASD `0.0.0316` (`/dev/dasdu1`, 2.3 GB) is used to add a fifth physical volume to
the existing `softdomvg` group, which hosts IBM software mount points:
 
```bash
vgextend softdomvg /dev/dasdu1
```
 
Create the new logical volume within `softdomvg`:
 
```bash
lvcreate -L 2342M --name optibm4lv softdomvg
```
 
> **NOTE:** LVM rounds the requested size up to the nearest physical extent boundary.
> `2342M` was rounded to `2.29 GB`. Use `--name` (two dashes), not `-name`.
 
---
 
### Step 8 - Create New Volume Groups for `intranet27`
 
Three new Volume Groups are needed, following the same naming convention as `intranet25`
and `intranet26`:
 
```bash
# Data VG - single 23 GB DASD
vgcreate intranet27datavg /dev/dasdv1
 
# Translog VG - single 7 GB DASD
vgcreate intranet27translogvg /dev/dasdw1
 
# Mail VG - three 23 GB DASDs (total ~67 GB, matching intranet25mailvg and intranet26mailvg)
vgcreate intranet27mailvg /dev/dasdx1 /dev/dasdy1 /dev/dasdz1
```
 
Verify the new VGs are visible:
 
```bash
vgs
```
 
---
 
### Step 9 - Create Logical Volumes
 
Create one LV per VG, sized to match the existing instances:
 
```bash
# Data LV - 22 GB
lvcreate -L 22G --name intranet27datalv intranet27datavg
 
# Translog LV - 7000 MB (VG capacity is ~6.88 GB; requesting 7 GB or 7042 MB exceeds
# the available extents - use 7000M as a safe value)
lvcreate -L 7000M --name intranet27transloglv intranet27translogvg
 
# Mail LV - 66 GB
lvcreate -L 66G --name intranet27maillv intranet27mailvg
```
 
> **NOTE:** The translog VG (`intranet27translogvg`) has ~6.88 GB total. Requesting `7G`,
> `6.88G`, `7043M`, or `7042M` all fail because LVM requires a few extents for metadata.
> Use `7000M` to stay within the available extent count.
 
Verify all LVs:
 
```bash
lvs
```
 
---
 
### Step 10 - Create ReiserFS Filesystems
 
Format each new logical volume with ReiserFS (the filesystem used by existing instances
on this server):
 
```bash
mkreiserfs /dev/softdomvg/optibm4lv
mkreiserfs /dev/intranet27datavg/intranet27datalv
mkreiserfs /dev/intranet27translogvg/intranet27transloglv
mkreiserfs /dev/intranet27mailvg/intranet27maillv
```
 
Type `y` at the confirmation prompt for each. `mkreiserfs` will display the UUID assigned
to each volume.
 
---
 
### Step 11 - Update `/etc/fstab`
 
Back up the current `fstab` before editing:
 
```bash
cp /etc/fstab /etc/fstab-20110714
```
 
Add mount entries for each new logical volume. Use the LVM device mapper path
(`/dev/<vgname>/<lvname>`). Example entries following the pattern of existing instances:
 
```
/dev/intranet27datavg/intranet27datalv     /intranet27/data     reiserfs  defaults  0 0
/dev/intranet27translogvg/intranet27transloglv /intranet27/translog reiserfs defaults 0 0
/dev/intranet27mailvg/intranet27maillv     /intranet27/mail     reiserfs  defaults  0 0
```
 
> **NOTE:** Adjust mount point paths to match the actual Domino directory layout under
> `/intranet27/` before saving.
 
---
 
## Verification
 
After completing all steps, confirm the final storage layout:
 
```bash
# All target DASDs active
lsdasd
 
# All VGs present with correct sizes and 0 free space (fully allocated)
vgs
 
# All LVs present with expected sizes
lvs
```
 
Expected final VG state for the new instance:
 
| VG                    | PVs | LVs | Size    | Free    |
|-----------------------|-----|-----|---------|---------|
| `intranet27datavg`    | 1   | 1   | 22.49 G | ~504 MB |
| `intranet27mailvg`    | 3   | 1   | 67.48 G | ~1.48 G |
| `intranet27translogvg`| 1   | 1   |  6.88 G | ~40 MB  |
 
---
 
## Troubleshooting
 
| Symptom | Cause | Resolution |
|---|---|---|
| `Could not resolve hostname` on SSH | Host alias missing or `~/.ssh/config` unreadable | Fix file ownership (`chown user ~/.ssh/config`) and add host block |
| `Device 0.0.0XXX is already online` | `dasd_configure` called twice | Harmless; device is online - continue |
| `Insufficient free extents` on `lvcreate` | Requested size exceeds VG capacity after metadata | Reduce size by a few MB (e.g. use `7000M` instead of `7042M`) |
| `Volume group "X" not found` on `lvcreate` | Used `-name` (one dash) instead of `--name` | Always use `--name` for the LV name option |
| `pvcreate` on raw device (`/dev/dasdX`) | Tab-completion selected the whole disk | Use the partition (`/dev/dasdX1`) - the one with the `1` suffix |
 
