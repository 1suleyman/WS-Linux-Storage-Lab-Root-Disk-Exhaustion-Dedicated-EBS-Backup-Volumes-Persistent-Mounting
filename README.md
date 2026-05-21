# AWS Linux Storage Lab – Root Disk Exhaustion, Dedicated EBS Backup Volumes & Persistent Mounting

In this lab, I simulated a real-world Linux production incident where backup files slowly filled the EC2 root disk, causing operational risk and potential outages. I investigated disk usage, identified the root cause, attached a dedicated EBS backup volume, mounted it properly, migrated workloads safely, configured persistent mounts using `fstab`, and implemented production-grade backup safety checks. 

---

# 📋 Lab Overview

## 🎯 Goal:

* Simulate root disk exhaustion on an EC2 Linux server
* Investigate disk usage using Linux tools
* Understand `df` vs `du`
* Attach and configure a second EBS volume
* Understand Linux block devices and mount points
* Create and format a file system using XFS
* Mount EBS storage persistently using `/etc/fstab`
* Safely migrate backup workloads off the root volume
* Implement backup safety checks to avoid silent root disk exhaustion
* Simulate reboot and mount persistence validation
* Simulate operational failure scenarios

---

# 📚 Learning Outcomes

By the end of this lab, I understood how to:

* Diagnose root disk exhaustion in Linux
* Use `df`, `du`, `lsblk`, and `blkid`
* Understand mount points and Linux file systems
* Understand block storage vs network file shares
* Create and format file systems using XFS
* Mount EBS volumes correctly
* Configure persistent mounts using UUIDs
* Understand `/etc/fstab`
* Understand Linux PATH and executable behavior
* Prevent dangerous backup misconfigurations
* Simulate real operational storage failures safely
* Understand how Linux handles mounted vs unmounted directories

---

# 🛠 Step-by-Step Journey

---

# Step 1: Launch EC2 Instance

Created:

* Amazon Linux 2023 EC2 instance
* 20 GB root EBS volume

Connected using SSH:

```bash id="5n06m0"
chmod 400 "labec2.pem"

ssh -i "labec2.pem" ec2-user@<PUBLIC-IP>
```



---

# Step 2: Simulate Root Disk Exhaustion

Checked initial disk usage:

```bash id="tk8qv4"
df -h
```

## Initial State

| Filesystem | Used   | Available | Usage |
| ---------- | ------ | --------- | ----- |
| Root (`/`) | 1.7 GB | 19 GB     | 9%    |

Created backup directory:

```bash id="d8nh1n"
sudo mkdir -p /var/backups
```

## 💡 Important Concept

```bash id="q7lb4n"
mkdir -p
```

means:

* create parent directories automatically if missing



---

# Step 3: Generate Fake Backup Files

Created large backup files:

```bash id="whu9k5"
sudo fallocate -l 2G /var/backups/backup1.image
sudo fallocate -l 2G /var/backups/backup2.image
sudo fallocate -l 2G /var/backups/backup3.image
sudo fallocate -l 2G /var/backups/backup4.image
```

## 💡 What `fallocate` Actually Does

It:

* reserves disk space immediately
* without manually writing huge data streams

## 🧠 Mental Model

```text id="rj7m1d"
"Reserve me a 2 GB box right now."
```



---

# Step 4: Understand `df` vs `du`

Checked filesystem usage:

```bash id="5d1tk6"
df -h
```

Checked directory usage:

```bash id="yo5n2j"
sudo du -sh /var/backups/*
```

## 💡 Critical Difference

| Command | Purpose                             |
| ------- | ----------------------------------- |
| `df`    | Shows entire filesystem usage       |
| `du`    | Shows specific directory/file usage |

## 🧠 Mental Model

| Tool | Analogy                              |
| ---- | ------------------------------------ |
| `df` | How full is the apartment building?  |
| `du` | Which apartments use the most space? |



---

# Step 5: Understand `du -sh`

## Flags

| Flag | Meaning        |
| ---- | -------------- |
| `-s` | Summary only   |
| `-h` | Human readable |

Example:

```bash id="80tymx"
sudo du -sh /var/backups/*
```



---

# Step 6: Understand Mount Points

## 💡 Key Concept

A mount point:

* is NOT the file system itself
* it is the doorway into another file system

## 🧠 Mental Model

```text id="3l8qhy"
Linux filesystem = giant building hallway
Mount point = door
EBS volume = storage room behind the door
```

Before mounting:

```text id="q3e0nk"
/backups = normal root directory
```

After mounting:

```text id="8kz5x4"
/backups = access point into another EBS volume
```



---

# Step 7: Understand Why `du /` Can Mislead

## Problem

```bash id="52f2ya"
du -sh /
```

may traverse:

* multiple mounted EBS volumes

## Safer Approach

```bash id="6e39q4"
du -x /
```

## 💡 Important

`-x` means:

* stay within the same filesystem only



---

# Step 8: Attach Second EBS Volume

Created:

* 20 GB EBS volume

Attached it:

* to EC2 instance

## Important Discovery

EBS must exist:

* in the SAME Availability Zone as the EC2 instance

## Why?

EBS is:

* network-attached block storage
* accessed over AWS internal networking



---

# Step 9: Understand Block Storage vs File Shares

## 💡 Critical Difference

| Type        | Behavior           |
| ----------- | ------------------ |
| EBS         | Raw block device   |
| NFS/SMB/EFS | Shared file access |

## EBS Mental Model

```text id="rz5m8h"
USB hard drive attached to laptop
```

## Network Share Mental Model

```text id="8m3nvo"
Google Drive shared folder
```



---

# Step 10: Detect New Linux Block Device

Ran:

```bash id="3gg0l5"
lsblk
```

## 💡 Important Clarification

Linux does NOT inherently know:

* "this is an AWS EBS volume"

Linux only sees:

* block devices



---

# Step 11: Understand `lsblk` vs `df`

| Command | Shows                |
| ------- | -------------------- |
| `lsblk` | Block devices        |
| `df -h` | Mounted file systems |

## Important Insight

A block device may exist:

* without being mounted



---

# Step 12: Check Existing File System

Ran:

```bash id="ejqtnj"
sudo blkid
```

## 💡 Purpose

Checks:

* whether block devices already contain file systems

Example:

```bash id="3hzj0s"
sudo blkid /dev/xvdc
```

If empty:

* no file system exists



---

# Step 13: Understand Partitions vs File Systems

## 💡 Critical Difference

| Concept     | Purpose                     |
| ----------- | --------------------------- |
| Partition   | Divides disk into sections  |
| File System | Organizes files/directories |

## 🧠 Mental Model

Partition:

```text id="4g7jti"
Slicing pie into pieces
```

File system:

```text id="4q7zvv"
Filing system inside each slice
```



---

# Step 14: Format EBS Volume

Created XFS file system:

```bash id="x0y6bm"
sudo mkfs.xfs /dev/xvdc
```

Validated:

```bash id="bz6q6d"
lsblk -f
```

## 💡 Important

Formatting:

* CREATES the file system on the disk

It does NOT:

* attach the disk anywhere



---

# Step 15: Create Mount Point

Created:

```bash id="nlm36m"
sudo mkdir /backups
```

## Important Discovery

Before mounting:

```text id="yzvgo5"
/backups = normal root directory
```

After mounting:

```text id="f7lvj8"
/backups = doorway into EBS volume
```

## Important Linux Behavior

Mounting:

* hides underlying directory contents temporarily
* does NOT delete them



---

# Step 16: Mount EBS Volume

Mounted EBS:

```bash id="m7m6cb"
sudo mount /dev/xvdc /backups
```

Validated:

```bash id="f0m7jt"
df -h
```

Result:

```text id="7r2nmg"
/dev/xvdc mounted on /backups
```



---

# Step 17: Migrate Backup Files Safely

Copied backups:

```bash id="z4m5a7"
sudo cp -av /var/backups/* /backups/
```

## 💡 What `-av` Means

| Flag | Meaning      |
| ---- | ------------ |
| `-a` | Archive mode |
| `-v` | Verbose      |

Archive mode preserves:

* permissions
* ownership
* timestamps
* symlinks



---

# Step 18: Remove Old Root Backups

Deleted original backups:

```bash id="i4u8yd"
sudo rm -f /var/backups/*
```

Validated recovered space:

```bash id="m0ix1w"
df -h
```

Result:

* root disk usage dropped significantly



---

# Step 19: Configure Persistent Mounting

Retrieved UUID:

```bash id="n69zyl"
sudo blkid /dev/xvdc
```

## 💡 Important Discovery

UUID belongs primarily to:

* the FILE SYSTEM
* not merely the raw block device



---

# Step 20: Understand `/etc/fstab`

## 💡 Key Concept

```text id="6ev6wy"
/etc/fstab
```

=
Linux file system table

Used during boot:

* to automatically remount file systems

## Without It

After reboot:

```text id="6w9wqa"
/backups becomes normal root directory again
```

which can silently fill the root disk again.



---

# Step 21: Add Persistent Mount Entry

Added:

```text id="7d9n6w"
UUID=<UUID> /backups xfs defaults,nofail 0 0
```

## Important Options

| Option     | Meaning                              |
| ---------- | ------------------------------------ |
| `defaults` | Standard Linux mount behavior        |
| `nofail`   | Continue boot even if volume missing |
| `0`        | Disable dump backup utility          |
| `0`        | Disable automatic fsck boot checks   |



---

# Step 22: Understand `fsck`

## 💡 What `fsck` Does

Checks:

* file system consistency
* corruption
* interrupted writes
* metadata mismatches

## Typical Causes

* sudden power loss
* crashed operating system
* interrupted disk writes



---

# Step 23: Validate Mount Persistence

Unmounted:

```bash id="6q5f3q"
sudo umount /backups
```

Tested all mounts:

```bash id="cfm9r3"
sudo mount -a
```

## 💡 Important

`mount -a`
simulates:

* boot-time mounting behavior



---

# Step 24: Debug Incorrect UUID Configuration

## Problem

Incorrect UUID entered into `fstab`.

Result:

* mount failed silently

## Root Cause

Linux attempted to mount:

* non-existent UUID

## Fix

Corrected UUID:

* remounted successfully



---

# Step 25: Understand Linux PATH

Stored script in:

```text id="95crh5"
/usr/local/bin
```

## 💡 Important

Directories in PATH:

* allow commands to run globally

Example:

```bash id="5e9n5t"
test-backup.sh
```

instead of:

```bash id="a6p7zj"
./test-backup.sh
```



---

# Step 26: Create Backup Safety Script

Created:

```text id="e0zv6w"
/usr/local/bin/test-backup.sh
```

## Safety Check

```bash id="n7c9eo"
mountpoint -q "$BACKUP_DIR"
```

## 💡 Critical Production Safety Concept

If `/backups` is NOT mounted:

* abort backup immediately

Otherwise:

* Linux silently writes into root disk again

## 🧠 Real Production Failure Pattern

Missing mount:

```text id="8h8e6w"
/backups becomes ordinary root directory
```

Application:

* continues writing normally
* silently fills root disk again



---

# Step 27: Simulate Mount Failure

Unmounted:

```bash id="25rnqa"
sudo umount /backups
```

Ran backup script:

```bash id="htifgo"
sudo test-backup.sh
```

Result:

```text id="j8f5fx"
ERROR: /backups is not mounted
Aborting backup
```

## 💡 Success

Safety mechanism worked correctly.



---

# Step 28: Understand "Target Is Busy"

## Error

```text id="p8jg0u"
target is busy
```

## Cause

Shell was:

* currently inside `/backups`

Linux refused:

* to unmount active filesystem

## Fix

Changed directories:

```bash id="xmgj6k"
cd ~
```

then retried unmount.



---

# Step 29: Simulate Full Root Disk

Created massive files:

```bash id="w14m9q"
sudo fallocate -l 4G /var/backups/big.image
sudo fallocate -l 4G /var/backups/big2.image
sudo fallocate -l 2G /var/backups/big3.image
```

Result:

* root disk reached 99% utilization

Validated:

```bash id="bq6f8g"
df -h
```



---

# Step 30: Observe System Behavior Under Full Disk

Tested:

* `yum update`
* file creation
* reboot persistence

## Important Observation

System:

* slowed down
* but remained operational

## Final Validation

After reboot:

```bash id="2zw0ja"
df -h
```

confirmed:

* `/backups` mounted successfully
* persistence worked correctly



---

# ✅ Key Commands Summary

| Task                     | Command                    |
| ------------------------ | -------------------------- |
| Show filesystem usage    | `df -h`                    |
| Show directory usage     | `du -sh`                   |
| Show block devices       | `lsblk`                    |
| Show filesystem metadata | `blkid`                    |
| Format XFS filesystem    | `mkfs.xfs`                 |
| Mount filesystem         | `mount /dev/xvdc /backups` |
| Unmount filesystem       | `umount /backups`          |
| Test fstab mounts        | `mount -a`                 |
| Create mount point       | `mkdir /backups`           |
| Copy backups safely      | `cp -av`                   |
| Create large files       | `fallocate -l`             |
| Check mount validity     | `mountpoint -q`            |
| Edit fstab               | `/etc/fstab`               |
| Show filesystem UUID     | `blkid /dev/xvdc`          |

---

# 💡 Notes / Tips

* `df` shows filesystem usage; `du` shows directory/file usage
* Mounted EBS volumes become part of the Linux directory tree
* Mount points are doors into other file systems
* Formatting creates a file system; mounting exposes it
* UUIDs are safer than device names for persistent mounts
* Missing mounts can silently redirect writes to the root disk
* `nofail` prevents EC2 boot failures if EBS is unavailable
* Linux block devices are not automatically mounted
* Always validate mounts using `mount -a`
* Safety checks prevent silent production failures
* EBS is network-attached block storage, not local server disks
* `target is busy` often means a process is still using the mount
* Root disk exhaustion can severely degrade Linux performance
