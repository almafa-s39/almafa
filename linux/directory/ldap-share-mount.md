<!-- 
---
title: "# LDAP login and automount"
author: "Gergő Téringer"
---
 -->

# LDAP login and automount

This document provides administrative procedures for integrating a Debian 13 (Trixie) system with an LDAP directory using the System Security Services Daemon (SSSD). It also covers configuring Pluggable Authentication Modules (PAM) to automatically mount CIFS network shares as user home directories upon login.

> [!WARNING]
> This setup uses **StartTLS** to authenticate with the provider. Ensure that you trust the root certificate for your LDAP provider by placing the CA certificate in `/usr/local/share/ca-certificates/` and running `update-ca-certificates`.

## 1. SSSD Package Installation and Core Configuration

The `sssd` service manages LDAP authentication and caches credentials for offline access. The `sssd.conf` file must be created manually and requires strict ownership and permission settings to start securely.

```bash
# Install SSSD and LDAP utilities
apt install sssd ldap-utils
```

**Command Breakdown & Explanation:**

- `apt install sssd ldap-utils`: Installs the System Security Services Daemon for authentication routing and the standard LDAP client tools (such as `ldapsearch`) for testing directory connectivity.

### 1.1 Configuring SSSD and PAM Home Directory Creation

```bash
# Create the SSSD configuration file
cat << 'EOF' > /etc/sssd/sssd.conf
[sssd]
config_file_version = 2
domains = lego.dk
services = nss,pam

[domain/lego.dk]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://HQ-DC.billund.lego.dk
cache_credentials = True
ldap_search_base = dc=lego,dc=dk
EOF

# Enforce strict ownership and permissions on the configuration file
chown root:root /etc/sssd/sssd.conf
chmod 600 /etc/sssd/sssd.conf

# Restart the SSSD service to apply configurations
systemctl restart sssd

# Enable automatic home directory creation in PAM
pam-auth-update --enable mkhomedir
```

**Command Breakdown & Explanation:**

- `/etc/sssd/sssd.conf`: Defines the directory domain (`lego.dk`), upstream LDAP server URI (`ldap://HQ-DC.billund.lego.dk`), and the domain search base (`dc=lego,dc=dk`).
- `chown root:root` and `chmod 600`: SSSD will explicitly refuse to start if its configuration file is readable by standard users, as it contains sensitive domain connection parameters.
- `systemctl restart sssd`: Restarts the daemon to parse the newly created configuration file.
- `pam-auth-update --enable mkhomedir`: Reconfigures the system PAM stack to automatically construct the `/home/<username>` directory skeleton upon the user's first successful login.

## 2. CIFS Automount Configuration using PAM Mount

To dynamically map a remote network share to the local home directory during authentication, you must install `libpam-mount` and configure its XML ruleset.

```bash
# Install PAM mount and CIFS utilities
apt install libpam-mount cifs-utils
```

**Command Breakdown & Explanation:**

- `apt install libpam-mount cifs-utils`: Installs the PAM module capable of executing mount commands during the login session lifecycle, along with the kernel utilities required to negotiate CIFS (SMB) filesystems.

### 2.1 Editing PAM Mount XML Configuration

The PAM mount configuration is maintained in `/etc/security/pam_mount.conf.xml`. You must uncomment the local users configuration tag and define the CIFS volume mapping parameters inside the primary configuration structure.

```xml
<!-- Locate and uncomment this line to allow user-level configurations if needed -->
<lusersconf name=".pam_mount.conf.xml" />

<!-- Add the CIFS volume definition inside the <pam_mount> root tag -->
<pam_mount>
  <debug enable="0" />
  <volume
      options="nodev,nosuid,nofail,nobrl,cache=none,noserverino"
      uid="%(USER)"
      path="%(USER)"
      mountpoint="/home/%(USER)"
      server="HQ-DC.billund.lego.dk"
      fstype="cifs"
  />
</pam_mount>
```

**Command Breakdown & Explanation:**

- `<lusersconf>`: Permits individual users to maintain a personal `.pam_mount.conf.xml` file.
- `<volume>`: Defines the automated mount action triggered by the PAM session phase.
- `options`: Hardens the mount by preventing device execution (`nodev`), ignoring setuid bits (`nosuid`), preventing boot hangs if unavailable (`nofail`), disabling byte-range locks (`nobrl`), and bypassing server inode generation (`noserverino`).
- `uid="%(USER)"`: Dynamically assigns filesystem ownership to the logging-in LDAP user.
- `path="%(USER)"`: Specifies the remote share path/folder on the server matching the username.
- `mountpoint="/home/%(USER)"`: Sets the local destination directory.
- `fstype="cifs"`: Instructs the mount command to utilize the SMB/CIFS protocol.

## 3. Verification and Troubleshooting

> [!NOTE]
> Validate the SSSD daemon status, LDAP user resolution, and PAM configuration integrity on Debian 13 using standard administrative commands.

### 3.1 Verify SSSD service status

**Command:** `systemctl status sssd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/sssd.service; enabled)`

### 3.2 Verify SSSD configuration file permissions

**Command:** `stat -c "%U:%G %a" /etc/sssd/sssd.conf`

**What it checks and variables to look for:**

- **Ownership and Permissions**: Must output exactly `root:root 600`

### 3.3 Verify LDAP user resolution via NSS

**Command:** `getent passwd <ldap_username>`

**What it checks and variables to look for:**

- **Output line**: Must return the user's POSIX account details from the LDAP directory (e.g., `username:*:10000:10000:User Name:/home/username:/bin/bash`)

### 3.4 Verify PAM mkhomedir configuration

**Command:** `grep "pam_mkhomedir.so" /etc/pam.d/common-session`

**What it checks and variables to look for:**

- **Module presence**: Must output a line containing `session optional pam_mkhomedir.so`

<!-- Created by: Gergő Téringer, 2026 -->