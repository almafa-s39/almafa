<!-- 
---
title: "Samba"
author: "Gergő Téringer"
---
 -->
# Samba

This document provides administrative procedures for installing, configuring, and managing a Samba file server on Debian 13 (Trixie). It covers user and group provisioning, directory permissions, public and private share definitions in `smb.conf`, and custom dynamic home directory auto-creation.

> [!NOTE]
> Samba implements the Server Message Block (SMB) protocol, enabling file and print sharing services for Windows, Linux, and macOS clients. On Debian 13, main configurations reside in `/etc/samba/smb.conf`.

## 1. Samba Package Installation and Service Management

To enable SMB network share hosting, install the core `samba` package from the repository binaries.

```bash
# Install the core Samba file server package
apt install samba
```

**Command Breakdown & Explanation:**

- `apt install samba`: Installs the Samba suite, including the SMB daemon (`smbd`), the NetBIOS name server daemon (`nmbd`), and basic client tools.

## 2. Local System User and Samba Account Management

Samba utilizes standard Linux system user accounts for underlying file system permissions, but manages authentication passwords using its own internal database (`passdb.tdb`).

> [!IMPORTANT]
> A user must exist as a valid Linux system user before they can be added to the Samba authentication database.

### 2.1 Provisioning System Users and Groups

```Bash
# Create non-interactive local system users without home directories
useradd -M -s /sbin/nologin sambauser
useradd -M -s /sbin/nologin jamie

# Create a dedicated group for share access and assign sambauser
groupadd smbshare
usermod -aG smbshare sambauser
```

**Command Breakdown & Explanation:**

- `useradd -M -s /sbin/nologin`: Creates a system user account without creating a default home directory (`-M`) and sets the shell to non-interactive (`-s /sbin/nologin`), restricting local terminal logins.
- `groupadd smbshare`: Provisions a system group used to regulate group-based file access.
- `usermod -aG smbshare sambauser`: Appends (`-aG`) `sambauser` to the `smbshare` supplementary group.

### 2.2 Registering and Enabling Samba Passwords

```bash
# Register and enable Samba access for sambauser
smbpasswd -a sambauser
smbpasswd -e sambauser

# Register and enable Samba access for jamie
smbpasswd -a jamie
smbpasswd -e jamie
```

**Command Breakdown & Explanation:**

- `smbpasswd -a <username>`: Adds the specified user account to the Samba password database and prompts for an SMB authentication password.
- `smbpasswd -e <username>`: Enables an existing account in the Samba database, allowing client connections.

## 3. Directory Provisioning and Share Configuration

File access is governed by the combination of local POSIX file system permissions and Samba share definitions in `/etc/samba/smb.conf`.

### 3.1 Creating Directory Structure and Permissions

```Bash
# Create and set permissions for the public share directory
mkdir -p /smb/public
chmod 2777 /smb/public
chown sambauser:smbshare /smb/public

# Create and set permissions for the private share directory
mkdir -p /smb/private
chmod 2770 /smb/private
chown jamie:jamie /smb/private
```

**Command Breakdown & Explanation:**

- `mkdir -p`: Creates parent directories as needed without throwing errors.
- `chmod 2777`: Grants read, write, and execute permissions to everyone while enforcing the SGID bit (`2`), ensuring new files created within the directory inherit the group ownership of `/smb/public`.
- `chmod 2770`: Restricts directory access strictly to the owner and group members (read, write, execute), denying all permissions to world/others.
- `chown`: Assigns specific POSIX user and group ownership to the target paths.

### 3.2 Configuring Global and Custom Shares in smb.conf

Append the share definitions to the end of `/etc/samba/smb.conf`. This includes public read-only access with group-write permissions, an isolated private share for `jamie`, and a dynamic `[homes]` section that automatically creates user directories on first logon.

> [!TIP]
> The `root preexec` directive allows Samba to execute root-level system commands before a client connects to a share, making it ideal for automatic directory creation.

```ini
[public]
  path = /smb/public
  read only = yes
  guest ok = yes
  writeable = no
  force user = nobody
  force group = nogroup
  create mask = 0777
  directory mask = 0777
  write list = @smbshare

[private]
  path = /smb/private
  valid users = jamie
  guest ok = no
  writeable = yes
  create mask = 0770
  directory mask = 0770

[homes]
   comment = Home directories
   browseable = no
   read only = no
   create mask = 0700
   directory mask = 0700
   valid users = %S
   path = /share/users/%S
   root preexec = bash -c 'mkdir -p /share/users/%S; chown %U:nogroup /share/users/%U; chmod 700 /share/users/%U'
```

```bash
# Restart the Samba service to load the new share configuration
systemctl restart smbd
```

**Command Breakdown & Explanation:**

- `guest ok = yes`: Permits unauthenticated guest access to the share.
- `write list = @smbshare`: Overrides `read only = yes` for members of the `smbshare` group (`@`), granting them write access.
- `valid users = %S`: Restricts access strictly to the authenticated user matching the share name session variable (`%S`).
- `path = /share/users/%S`: Dynamically maps the home share path to a custom directory structure based on the username.
- `root preexec`: Executes a shell string as `root` prior to mounting the share, creating `/share/users/%S` on-the-fly and assigning owner permissions (`%U`).
- `systemctl restart smbd`: Reloads the configuration file and applies changes to active listeners.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the status of the Samba service, configuration syntax, active listening ports, and local share availability on Debian 13 using standard diagnostic tools.

### 4.1 Verify Samba smbd service running status

**Command:** `systemctl status smbd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/smbd.service; enabled)`

### 4.2 Verify Samba configuration file syntax using testparm

**Command:** `testparm -s`

**What it checks and variables to look for:**

- **Loaded services file**: Must state `Loaded services file OK.`
- **Role**: Must display `ROLE_STANDALONE` (or your target role) without structural parsing errors

### 4.3 Verify network listening state on SMB ports

**Command:** `ss -tuln | grep -E "139|445"`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:139` and `*:445` (or `0.0.0.0:139` and `0.0.0.0:445`)

### 4.4 Verify local SMB share enumeration

**Command:** `smbclient -L //127.0.0.1 -N`

**What it checks and variables to look for:**

- **Sharename**: Output list must include `public`, `private`, and default IPC share

**Command:** `smbclient -L //127.0.0.1 -N`

**What it checks and variables to look for:**

- **Sharename**: Output list must include `public`, `private`, and default IPC shares

<!-- Created by: Gergő Téringer, 2026 -->