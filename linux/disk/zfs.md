<!-- 
---
title: "zfs"
author: "Gergő Téringer"
---
 -->
# ZFS

ZFS Pool, Dataset, Volume, and Encryption Management Guide
Markdown

This document provides administrative procedures for deploying, managing, and troubleshooting ZFS on Debian 13 (Trixie). It covers package installation, pool and VDEV creation, dataset and volume (ZVOL) provisioning, snapshot management, and native ZFS encryption.

> [!NOTE]
> ZFS combines file system and volume management capabilities. Storage space is managed dynamically within centralized storage pools (`zpool`) composed of Virtual Devices (VDEVs).

## 1. ZFS Architecture and Package Installation

ZFS utilizes specialized abstractions to manage physical disks and logical containers:

- **Storage Pool (`zpool`)**: The top-level storage container built from one or more VDEVs striped together.
- **VDEV (Virtual Device)**: A logical group of physical disks providing redundancy (such as `mirror`, `raidz1`, or `raidz2`).
- **Dataset**: A filesystem created inside a pool that dynamically allocates space without fixed size restrictions.
- **ZVOL (ZFS Volume)**: A block device provisioned on top of a pool, allowing standard filesystems (such as ext4 or xfs) to be formatted on top.

> [!NOTE]
> On Debian 13, `zfsutils-linux` is located in the `contrib` repository component. Ensure `contrib` is included in your `/etc/apt/sources.list` before installing packages from your installation media.

```Bash
# Install kernel headers and ZFS userland utilities
apt install linux-headers-amd64 zfsutils-linux
```

**Command Breakdown & Explanation:**

- `apt install linux-headers-amd64 zfsutils-linux`: Installs Linux kernel C header files (required to compile or bind OpenZFS kernel modules) and the primary ZFS management CLI utilities (`zpool` and `zfs`).

## 2. ZFS Storage Pool Creation

> [!TIP]
> Always reference physical disks by their persistent path identifiers (`/dev/disk/by-path/`) rather than dynamic device names (such as `/dev/sda`), which can reorder upon system reboots.

```Bash
# List persistent disk path identifiers
ls -l /dev/disk/by-path/ | rev | cut -d' ' -f3 | rev

# Create a RAIDZ1 pool named 'pool1' across 4 physical drives
zpool create pool1 \
  raidz1 \
    pci-0000:03:00.0-scsi-0:0:1:0 \
    pci-0000:03:00.0-scsi-0:0:2:0 \
    pci-0000:03:00.0-scsi-0:0:3:0 \
    pci-0000:03:00.0-scsi-0:0:4:0

# Create a mirrored pool (RAID 10 equivalent) named 'pool2' across two mirror VDEVs
zpool create pool2 \
  mirror \
    pci-0000:03:00.0-scsi-0:0:1:0 \
    pci-0000:03:00.0-scsi-0:0:2:0 \
  mirror \
    pci-0000:03:00.0-scsi-0:0:3:0 \
    pci-0000:03:00.0-scsi-0:0:4:0
```

**Command Breakdown & Explanation:**

- `ls -l /dev/disk/by-path/...`: Parses and isolates persistent hardware paths assigned to physical drives.
- `zpool create pool1 raidz1`: Provisions a storage pool named `pool1` using a single RAIDZ1 VDEV (single parity, similar to RAID-5) across four drives.
- `zpool create pool2 mirror ... mirror ...`: Provisions a storage pool named `pool2` striped across two separate two-drive mirror VDEVs (similar to RAID-10).

## 3. Dataset and ZVOL Management

ZFS provides two distinct storage types: filesystems (Datasets) and block devices (ZVOLs).

### 3.1 Managing Datasets

Datasets inherit properties from the parent pool and automatically handle mount lifecycle state.

```Bash
# Create a local target directory and provision a mounted dataset
mkdir -p /data
zfs create -o mountpoint=/data pool1/data

# Destroy a dataset
zfs destroy pool1/data
```

**Command Breakdown & Explanation:**

- `zfs create -o mountpoint=/data pool1/data`: Provisions dataset `data` under `pool1` and sets its automatic mount point to `/data`. Mount persistence is managed internally by ZFS without requiring `/etc/fstab` entries.
- `zfs destroy pool1/data`: Permanently deletes the specified dataset and unmounts its filesystem.

### 3.2 Managing ZVOLs (Block Storage Volumes)

ZVOLs present raw block storage interfaces located under `/dev/zvol/<pool>/<volume>`.

```bash
# Provision a 4GB sparse (thinly-provisioned) ZVOL block device
zfs create -s -V 4GB pool1/volume1

# Format the ZVOL with ext4 and mount it manually
mkfs.ext4 /dev/zvol/pool1/volume1
mkdir -p /mnt/volume1
mount /dev/zvol/pool1/volume1 /mnt/volume1

# Unmount and destroy the ZVOL block volume
umount /dev/zvol/pool1/volume1
zfs destroy pool1/volume1
```

**Command Breakdown & Explanation:**

- `zfs create -s -V 4GB pool1/volume1`: Creates a block device volume named `volume1`. The `-s` flag enables sparse allocation (thin provisioning), consuming physical storage only as data is written.
- `mkfs.ext4 /dev/zvol/...`: Formats the raw ZVOL block interface with a standard Linux ext4 filesystem.
- `umount` & `zfs destroy`: Manual unmounting is required prior to volume deletion because traditional filesystem mounts created on ZVOLs are managed outside ZFS.

## 4. Snapshots and Native Encryption

ZFS includes built-in point-in-time snapshotting and dataset-level encryption.

### 4.1 Managing Snapshots

Snapshots are read-only point-in-time copies of datasets or volumes that consume space only as underlying blocks change.

```Bash
# Create a snapshot of the dataset
zfs snapshot pool1/data@2025-05-20

# Roll back the dataset to the snapshot state
zfs rollback pool1/data@2025-05-20

# Destroy the snapshot
zfs destroy pool1/data@2025-05-20
```

**Command Breakdown & Explanation:**

- `zfs snapshot`: Captures an instantaneous, zero-copy snapshot of dataset `pool1/data` named `@2025-05-20`.
- `zfs rollback`: Reverts dataset state back to the specified snapshot identifier, discarding modifications made after the snapshot was captured.
- `zfs destroy`: Removes the snapshot record and frees unreferenced data blocks.

### 4.2 Native ZFS Encryption and Key Management

Native encryption operates at the dataset or pool layer using AES ciphers.

```Bash
# Create an encrypted ZFS pool using a passphrase
zpool create \
  -O encryption=aes-256-gcm \
  -O keyformat=passphrase \
  encryptedpool \
  pci-0000:03:00.0-scsi-0:0:1:0

# Create a dataset on the encrypted pool (inherits encryption settings)
zfs create encryptedpool/securedataset

# Manually load the encryption key after a system reboot
zfs load-key encryptedpool

# Mount all available ZFS filesystems
zfs mount -a
```

> [!WARNING]
> Storing encryption passphrases in plain text files to enable automated mounting weakens security. Protect key files with strict file permissions (`600`).

```Bash
# Configure automated key loading using a local passphrase file
echo 'MySecretPassphrase123!' > /etc/passphrase
chown root:root /etc/passphrase
chmod 600 /etc/passphrase

# Create /etc/rc.local script for startup mounting
cat << 'EOF' > /etc/rc.local
#!/bin/bash
zfs mount -l -a < /etc/passphrase
EOF

# Make rc.local executable
chmod +x /etc/rc.local
```

**Command Breakdown & Explanation:**

- `-O encryption=aes-256-gcm -O keyformat=passphrase`: Configures the root pool dataset with AES-256-GCM encryption authenticated via interactive passphrase entry.
- `zfs load-key encryptedpool`: Prompts for or reads the passphrase required to decrypt key material in memory following a host reboot.
- `zfs mount -l -a < /etc/passphrase`: Loads encryption keys (`-l`) for all locked pools using standard input redirection before mounting (`-a`) active datasets.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate ZFS pool health, dataset configurations, volume usage, and encryption states on Debian 13 using native utilities.

### 5.1 Verify ZFS pool status and health

**Command:** `zpool status pool1`

**What it checks and variables to look for:**

- **state**: Must be `ONLINE`
- **scan**: Must show `none requested` or completed scrub statistics without errors
- **errors**: Must display `No known data errors`

### 5.2 Verify dataset and volume disk space usage

**Command:** `zfs get all pool1/volume1 | grep used`

**What it checks and variables to look for:**

- **used**: Displays total space allocated to the dataset or volume (e.g., `4.12G`)

### 5.3 Verify ZFS encryption properties

**Command:** `zfs get encryption encryptedpool`

**What it checks and variables to look for:**

- **PROPERTY**: Must display `encryption`
- **VALUE**: Must display `aes-256-gcm` (or chosen cipher algorithm, must not be `off`)

### 5.4 Verify loaded encryption keys

**Command:** `zfs get keystatus encryptedpool`

**What it checks and variables to look for:**

- **PROPERTY**: Must display `keystatus`
- **VALUE**: Must display `available`

<!-- Created by: Gergő Téringer, 2026 -->