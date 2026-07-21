<!-- 
---
title: "Disk management, RAID"
author: "Gergő Téringer"
---
 -->
# Disk management, RAID

Disk Management and Software RAID Configuration Guide
Markdown

This document provides administrative instructions for initializing physical disks, provisioning standard NTFS storage partitions, converting basic disks to dynamic storage, and configuring software-based RAID volumes (RAID 0, RAID 1, and RAID 5) on Windows systems.

> [!NOTE]
> While `diskpart` dynamic disk features remain supported for legacy compatibility, modern deployments on Windows Server 2025 and Windows 11 recommend utilizing **Storage Spaces** (`New-StoragePool` and `New-VirtualDisk`) for software storage management and redundancy.

## 1. Disk Initialization and Basic Partition Provisioning

> [!IMPORTANT]
> Initializing a drive clears its partition table structure. Ensure the correct target disk index (`n`) is identified before executing destructive initialization or partition commands.

```PowerShell
# PowerShell initialization sequence
Get-Disk
Initialize-Disk -Number <DISK_NUMBER> -PartitionStyle GPT

# Interactive Diskpart sequence for basic volume creation
diskpart
list disk
select disk <DISK_NUMBER>
create partition primary
format fs=ntfs quick
assign letter=K
exit
```

**Command Breakdown & Explanation:**

- `Get-Disk`: Displays physical disks attached to the system along with their disk number, friendly name, operational status, and partition style.
- `Initialize-Disk -Number <DISK_NUMBER> -PartitionStyle GPT`: Prepares a raw disk using GUID Partition Table (GPT) partitioning.
- `diskpart`: Launches the native Windows command-line disk management shell.
- `list disk`: Enumerates all available physical disks within the `diskpart` console.
- `select disk <DISK_NUMBER>`: Sets active focus to the specified target disk index.
- `create partition primary`: Allocates the unpartitioned disk space into a primary raw partition.
- `format fs=ntfs quick`: Performs a rapid format on the partition using the NTFS file system.
- `assign letter=K`: Binds drive letter `K:` to the formatted partition.
- `exit`: Terminates the interactive `diskpart` session.

## 2. Dynamic Disk Conversion and Software RAID Provisioning

> [!CAUTION]
> Converting a basic disk to a dynamic disk is an irreversible operation without destroying existing volumes. Reverting a dynamic disk back to basic requires deleting all existing volumes, leading to complete data loss.

```PowerShell
# Convert target physical disks to Dynamic Disks
diskpart
list disk
select disk <DISK_NUMBER>
convert dynamic

# Provision RAID 0 (Striped Volume - Requires minimum 2 disks)
create volume stripe disk=1,2 size=10000

# Provision RAID 1 (Mirrored Volume - Requires minimum 2 disks)
create volume mirror disk=1,2 size=50000

# Provision RAID 5 (Striped with Parity Volume - Requires minimum 3 disks)
create volume raid disk=1,2,3 size=100000

# Assign drive letter and format the newly created RAID volume
list volume
select volume <VOLUME_NUMBER>
assign letter=E
format fs=ntfs quick
exit
```

**Command Breakdown & Explanation:**

- `convert dynamic`: Converts the designated basic disk into a dynamic disk, enabling multi-disk software volume layouts.
- `create volume stripe disk=1,2 size=10000`: Creates a software RAID 0 (Striped) volume across Disk 1 and Disk 2 allocating 10,000 MB per disk (20,000 MB total capacity) to increase throughput without data redundancy.
- `create volume mirror disk=1,2 size=50000`: Creates a software RAID 1 (Mirrored) volume across Disk 1 and Disk 2 allocating 50,000 MB per disk, offering 1-to-1 data redundancy against single-disk failure.
- `create volume raid disk=1,2,3 size=100000`: Creates a software RAID 5 volume across Disks 1, 2, and 3 with distributed parity, enabling fault tolerance against a single drive failure.
- `list volume`: Lists all active volumes and their index numbers.
- `select volume <VOLUME_NUMBER>`: Sets focus to the designated RAID volume.
- `assign letter=E`: Binds drive letter `E:` to the active software RAID volume.
- `format fs=ntfs quick`: Executes a fast format on the software RAID array using the NTFS file system.

## 3. Verification and Troubleshooting

> [!NOTE]
> Execute these verification cmdlets in an elevated PowerShell terminal or `diskpart` shell on Windows 11 or Windows Server 2025 to validate disk health, volume status, and dynamic array integrity.

### 3.1 Verify physical disk operational status and partition style

**Command:** `Get-Disk | Select-Object Number, FriendlyName, OperationalStatus, PartitionStyle, IsOffline`

**What it checks and variables to look for:**

- **OperationalStatus**: Must be `Online`
- **PartitionStyle**: Must be `GPT` or `MBR`
- **IsOffline**: Must be `False`

### 3.2 Verify volume health, file system, and drive letter assignment

**Command:** `Get-Volume | Select-Object DriveLetter, FileSystem, HealthStatus, OperationalStatus`

**What it checks and variables to look for:**

- **DriveLetter**: Must be `K` or `E`
- **FileSystem**: Must be `NTFS`
- **HealthStatus**: Must be `Healthy`
- **OperationalStatus**: Must be `OK`

### 3.3 Verify software RAID volume layout and dynamic disk status

**Command:** `diskpart` -> `list volume`

**What it checks and variables to look for:**

- **Type**: Must be `Stripe`, `Mirror`, or `RAID-5`
- **Status**: Must be `Healthy`
- **Info**: Must show valid mount state or blank (must not show `At Risk` or `Failed`)

<!-- Created by: Gergő Téringer, 2026 -->