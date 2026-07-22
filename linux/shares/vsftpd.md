<!-- 
---
title: "VsFTPD"
author: "Gergő Téringer"
---
 -->
# VsFTPD

This document details administrative procedures for installing, securing, and configuring Very Secure FTP Daemon (VSFTPD) on Debian 13 (Trixie). It covers user provisioning, directory permissions, SSL/TLS encryption settings, user list access control, and client verification testing.

> [!NOTE]
> VSFTPD is a lightweight and secure FTP server for Unix-like systems. Enforcing SSL/TLS encryption protects credentials and data transfers from plain-text exposure.

## 1. VSFTPD Package Installation and User Setup

To install VSFTPD and set up local FTP users, create the system user account and append it to the dedicated VSFTPD user list file.

> [!IMPORTANT]
> The `/etc/vsftpd.userlist` file works in conjunction with `userlist_deny=NO` to explicitly restrict FTP access solely to authorized accounts.

```Bash
# Install VSFTPD package
apt install vsftpd

# Create a new local user for FTP access
adduser <username>

# Add the authorized user to the VSFTPD user list
echo <username> | tee -a /etc/vsftpd.userlist
```

**Command Breakdown & Explanation:**

- `apt install vsftpd`: Installs the Very Secure FTP Daemon package.
- `adduser <username>`: Provisions a new local system user account.
- `echo <username> | tee -a /etc/vsftpd.userlist`: Appends the new username to the user list file used for access filtering.

## 2. FTP Directory Structure and Permission Assignment

When `chroot_local_user=YES` is enabled, VSFTPD prevents users from accessing directories outside their designated root directory. The chroot root directory itself must not be writeable by the user, requiring a dedicated writeable subdirectory for file uploads.

```Bash
# Create unwriteable base FTP root directory
mkdir -p /var/ftp

# Set ownership and remove write permissions from base directory
chown nobody:nogroup /var/ftp
chmod a-w /var/ftp

# Create writeable sub-directory for user uploads
mkdir -p /var/ftp/upload
chown <username>:<username> /var/ftp/upload
chmod 777 /var/ftp/upload
```

**Command Breakdown & Explanation:**

- `mkdir -p /var/ftp`: Creates the secure root path for local FTP storage.
- `chown nobody:nogroup /var/ftp`: Assigns root folder ownership to unprivileged user/group entities.
- `chmod a-w /var/ftp`: Strips write access from all users on the top-level root folder to comply with VSFTPD chroot security constraints.
- `mkdir -p /var/ftp/upload`: Provisions an internal upload directory where user files can be created and modified.
- `chown <username>:<username> /var/ftp/upload`: Sets POSIX ownership of the upload directory to the target user.
- `chmod 777 /var/ftp/upload`: Assigns full read, write, and execute permissions to the upload subdirectory.

## 3. Server Configuration and SSL/TLS Hardening

Configure `/etc/vsftpd.conf` to enable local authentication, restrict user directories via chroot, enforce explicit SSL/TLS encryption for logins and data transfers, and restrict client connections via `/etc/vsftpd.userlist`.

```Ini
# Listening Socket Options
listen=NO
listen_ipv6=YES

# Access Control and Local User Settings
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES

# Chroot Isolation Options
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd

# SSL / TLS Encryption Settings
rsa_cert_file=/etc/ssl/certs/pa-file.crt
rsa_private_key_file=/etc/ssl/private/pa-file.key
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH

# Directory and User List Configuration
user_sub_token=$USER
local_root=/var/ftp
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
#allow_writeable_chroot=YES
```

```Bash
# Restart the VSFTPD service to apply configuration changes
systemctl restart vsftpd
```

**Command Breakdown & Explanation:**

- `listen_ipv6=YES`: Configures VSFTPD to listen on IPv6 sockets while handling IPv4 clients.
- `anonymous_enable=NO`: Disallows unauthenticated anonymous FTP access.
- `local_enable=YES`: Permits local system users to log in via FTP.
- `write_enable=YES`: Enables write commands such as file uploads and directory creation.
- `chroot_local_user=YES`: Restricts local users to their designated home/root directories upon login.
- `force_local_data_ssl=YES` & `force_local_logins_ssl=YES`: Mandates TLS encryption for authentication and data channels.
- `userlist_deny=NO`: Changes user list behavior to act as an allowlist, permitting access only to accounts listed in `/etc/vsftpd.userlist`.
- `systemctl restart vsftpd`: Restarts the daemon to apply configuration changes.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate VSFTPD daemon status, active network ports, and client connectivity on Debian 13 using standard diagnostic tools.

### 4.1 Verify VSFTPD service running status

**Command:** `systemctl status vsftpd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/vsftpd.service; enabled)`

### 4.2 Verify network listening state on FTP command port 21

**Command:** `ss -tuln | grep :21`

**What it checks and variables to look for:**

- **State**: Must display `LISTEN`
- **Local Address:Port**: Must display `*:21` or `[::]:21`

### 4.3 Verify remote FTP connection using lftp

**Command:** `lftp -u <username> <ftp_server>`

**What it checks and variables to look for:**

- **Password prompt**: Must prompt for user password and connect successfully
- **Directory navigation**: Must allow listing and writing inside the `/upload` directory

<!-- Created by: Gergő Téringer, 2026 -->