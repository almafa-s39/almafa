<!-- 
---
title: "gnome-login-message"
author: "Gergő Téringer"
---
 -->

# GNOME Login Screen Banner Configuration Guide

This document provides administrative procedures for configuring a custom text banner on the GNOME Display Manager (GDM) login screen on Debian 13 (Trixie). This is achieved by utilizing the `dconf` configuration system to modify system-level GDM greeter settings.

> [!NOTE]
> The GNOME login screen runs under a dedicated `gdm` system user. Applying custom settings requires creating system-wide database overrides rather than modifying individual user profiles.

## 1. GDM dconf Profile Configuration

To apply custom configuration keys to the login screen, you must first define a `dconf` profile specifically for GDM. This profile instructs the system to read both the standard user database and the designated system-wide GDM database.

```Bash
# Create the necessary directory structure for the dconf profile
mkdir -p /etc/dconf/profile

# Create the GDM dconf profile
cat << 'EOF' > /etc/dconf/profile/gdm
user-db:user
system-db:gdm
file-db:/usr/share/gdm/greeter-dconf-defaults
EOF
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `user-db:user`: Specifies the default per-user configuration database.
- `system-db:gdm`: Specifies the custom system-level database for the GDM user context.
- `file-db:/usr/share/gdm/greeter-dconf-defaults`: Points to the fallback default GDM settings provided natively by the Debian package.

## 2. Banner Message Keyfile Creation

Once the profile is defined, you must create a machine-wide keyfile containing the specific GSettings overrides for the `org.gnome.login-screen` schema.

> [!TIP]
> Ensure the directory structure exists before creating the keyfile. The banner text can be customized to include standard system warning banners, legal notices, or support contact information.

```Bash
# Create the directory for the system-wide GDM database keyfiles
mkdir -p /etc/dconf/db/gdm.d/

# Create the keyfile enabling and defining the banner message
cat << 'EOF' > /etc/dconf/db/gdm.d/01-banner-message
[org/gnome/login-screen]
banner-message-enable=true
banner-message-text='Type the banner message here.'
EOF
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `[org/gnome/login-screen]`: Targets the specific dconf schema controlling the login screen graphical interface.
- `banner-message-enable=true`: A boolean key that activates the banner element on the user interface.
- `banner-message-text='...'`: A string key containing the exact text to display above the user selection list.

## 3. Database Compilation and Application

Changes made to text-based keyfiles in `/etc/dconf/db/` do not take effect immediately. The `dconf` engine requires compiling these text files into a binary database.

> [!WARNING]
> Restarting the GDM service will immediately log out any active desktop sessions.

```Bash
# Compile the updated keyfiles into the system database
dconf update

# Restart the GNOME Display Manager to apply the banner to the greeter
systemctl restart gdm3
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `dconf update`: Reads all configuration files inside `/etc/dconf/db/gdm.d/` and compiles them into the binary database used by the GNOME environment.
- `systemctl restart gdm3`: Restarts the GNOME Display Manager daemon, forcing a reload of the greeter UI with the newly configured banner text.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the compiled dconf settings and the GNOME Display Manager service status on Debian 13 using standard administrative tools.

### 4.1 Verify compiled dconf settings for GDM

**Command:** `su -s /bin/bash -c "gsettings get org.gnome.login-screen banner-message-enable" gdm`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Output**: Must return `true`

### 4.2 Verify banner message string compilation

**Command:** `su -s /bin/bash -c "gsettings get org.gnome.login-screen banner-message-text" gdm`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Output**: Must return the exact string defined in your keyfile (e.g., `'Type the banner message here.'`)

### 4.3 Verify GDM service running status

**Command:** `systemctl status gdm3`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/gdm3.service; enabled)`

<!-- Created by: Gergő Téringer, 2026 -->