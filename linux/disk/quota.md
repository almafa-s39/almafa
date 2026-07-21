<!-- 
---
title: "Quota"
author: "Gergő Téringer"
---
 -->
# Quota

This document provides administrative procedures for enabling, configuring, and monitoring filesystem disk quotas on Debian 13 (Trixie). It covers package installation, ext4 quota filesystem feature updates, `/etc/fstab` configuration, setting user limits and grace periods, and reporting quota usage.

> [!NOTE]
> Modern ext4 filesystems support internal quota management via the `quota` feature flag, eliminating the need for legacy `aquota.user` or `aquota.group` files.

## 1. Package Installation and Ext4 Feature Activation

> [!IMPORTANT]
> Modifying filesystem feature flags using `tune2fs` requires unmounting the target block device first to prevent data corruption.

```Bash
# Install the quota management utilities
apt install quota

# Unmount the target volume before updating filesystem features
umount /dev/md0

# Enable the internal quota feature flag on the ext4 filesystem
tune2fs -O quota /dev/md0
```

**Command Breakdown & Explanation:**

- `apt install quota`: Installs user and group quota management tools including `setquota`, `repquota`, and `quota`.
- `umount /dev/md0`: Unmounts the target storage volume (such as a RAID array or disk partition) so filesystem flags can be safely modified.
- `tune2fs -O quota /dev/md0`: Enables the built-in ext4 quota feature flag, allowing the kernel to track block and inode usage natively.

## 2. Mount Options and Systemd Configuration

To enforce quotas across system reboots, mount options must be defined in `/etc/fstab`.

> [!TIP]
> The `usrquota` and `grpquota` mount options instruct the kernel to enforce user and group quota limits on the mounted filesystem.

```Bash
# Add or update mount options in /etc/fstab (Example entry):
# /dev/sda1       /home   ext4    defaults,usrquota,grpquota      0       2

# Reload systemd unit files to recognize fstab changes and remount all filesystems
systemctl daemon-reload
mount -a
```

**Command Breakdown & Explanation:**

- `usrquota,grpquota`: Options added to the 4th field of `/etc/fstab` to enable kernel-level tracking and enforcement for users and groups.
- `systemctl daemon-reload`: Re-initializes systemd generator units to parse the updated `/etc/fstab` file.
- `mount -a`: Mounts or remounts all filesystems listed in `/etc/fstab`.

## 3. Assigning Quota Limits and Grace Periods

Quotas consist of **soft limits** (warnings triggered when exceeded) and **hard limits** (strict enforcement blocking further writes). Grace periods define how long a user may remain above their soft limit before it behaves as a hard limit.

```Bash
# Set user block and inode quotas (Limits are specified in KB blocks)
setquota -u <username> <soft_limit_KB> <hard_limit_KB> 0 0 <mountpoint>

# Set global grace periods for block and inode soft limit overages
setquota -t <block_grace_time> <inode_grace_time> <mountpoint>
```

**Command Breakdown & Explanation:**

- `setquota -u <username>`: Configures quota limits for a specific user.
- `<soft_limit_KB> <hard_limit_KB>`: Specifies the soft and hard storage space limits in 1 KB blocks (e.g., `500000` for ~500 MB).
- `0 0`: Sets the soft and hard inode (file count) limits to `0` (unlimited).
- `<mountpoint>`: Target directory path where the quota-enabled filesystem is mounted (e.g., `/home`).
- `setquota -t`: Modifies global grace periods (e.g., `7days` or `604800` seconds) before soft limits trigger write rejections.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate user quota usage, grace periods, and mount options on Debian 13 using standard administrative commands.

### 4.1 Verify individual user quota status

**Command:** `quota -vs <username>`

**What it checks and variables to look for:**

- **space**: Current block usage relative to soft and hard limits
- **quota**: Configured soft limit in human-readable format
- **limit**: Configured hard limit in human-readable format

### 4.2 Verify system-wide filesystem quota usage

**Command:** `repquota -s /home | grep -i "Block"`

**What it checks and variables to look for:**

- **Block Limits**: Displays current usage, soft limits, and hard limits for all users on the mount point
- **Grace Time**: Shows remaining grace period duration for users currently exceeding soft limits

### 4.3 Verify active filesystem mount options

**Command:** `findmnt /home`

**What it checks and variables to look for:**

- **OPTIONS**: Must contain `usrquota` and `grpquota`
- **FSTYPE**: Must display `ext4`

<!-- Created by: Gergő Téringer, 2026 -->