<!-- 
---
title: "FreeRADIUS LDAP Authentication"
author: "Gergő Téringer"
---
 -->
# FreeRADIUS LDAP Authentication

This document provides administrative instructions for configuring FreeRADIUS on Debian 13 (Trixie) to authenticate network access requests (such as OpenVPN logins) against a central LDAP directory. This lightweight Identity Services Engine (ISE) architecture centralizes authorization and authentication pipelines across network endpoints.

> [!NOTE]
> FreeRADIUS stores configuration files under `/etc/freeradius/3.0/`. Enabling LDAP authentication requires linking the LDAP module and configuring the default virtual server pipeline.

## 1. Package Installation and Base Environment Cleanup

> [!IMPORTANT]
> Removing the default `inner-tunnel` site configuration prevents startup conflicts and unneeded EAP processing when operating a straightforward PAP/LDAP authentication architecture.

```bash
# Update repositories and install FreeRADIUS with LDAP extensions
apt install freeradius freeradius-ldap freeradius-utils

# Remove the default inner-tunnel virtual server link
rm -f /etc/freeradius/3.0/sites-enabled/inner-tunnel
```

**Command Breakdown & Explanation:**

- `apt install freeradius freeradius-ldap freeradius-utils`: Installs the FreeRADIUS core daemon, LDAP integration module (`rlm_ldap`), and CLI diagnostic tools (`radtest`).
- `rm -f /etc/freeradius/3.0/sites-enabled/inner-tunnel`: Removes the default EAP inner-tunnel virtual server binding to streamline packet processing.

## 2. Default Virtual Server LDAP Pipeline Configuration

To process incoming authentication requests via LDAP, update the `default` virtual server policy in `sites-available/default`. This forces FreeRADIUS to check user credentials against the LDAP directory during the authorization phase and sets `Auth-Type := ldap`.

> [!TIP]
> Ensure the LDAP module symlink (`/etc/freeradius/3.0/mods-enabled/ldap`) is enabled and populated with valid LDAP bind credentials before restarting the service.

```bash
# Enable the LDAP module symlink if not already active
ln -sf /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/ldap

# Configure the default virtual server authentication pipeline
tee /etc/freeradius/3.0/sites-available/default > /dev/null << 'EOF'
server default {
    listen {
        type   = auth
        ipaddr = *
        port   = 1812
    }

    listen {
        type   = acct
        ipaddr = *
        port   = 1813
    }

    authorize {
        ldap
        if (ok || updated) {
            update control {
                Auth-Type := ldap
            }
        }
    }

    authenticate {
        Auth-Type LDAP {
            ldap
        }
    }

    accounting {
        ok
    }
}
EOF

# Ensure the default site is linked in sites-enabled
ln -sf /etc/freeradius/3.0/sites-available/default /etc/freeradius/3.0/sites-enabled/default
```

**Command Breakdown & Explanation:**

- `ln -sf .../mods-available/ldap .../mods-enabled/ldap`: Symlinks the LDAP module configuration into active runtime memory.
- `server default`: Defines the primary virtual server instance listening on UDP ports 1812 (auth) and 1813 (accounting).
- `authorize { ldap ... }`: Queries the LDAP database for the incoming user entity; if found (`ok` or `updated`), sets the internal control flag `Auth-Type := ldap`.
- `authenticate { Auth-Type LDAP { ldap } }`: Directs password verification to the `rlm_ldap` module driver.

## 3. Network Access Server Client Authorization

Network Access Servers (NAS)—such as an OpenVPN gateway—must be defined in `clients.conf` with a designated IP address and shared secret before sending authentication packets to FreeRADIUS.

```bash
# Append OpenVPN server client entry to clients.conf
tee -a /etc/freeradius/3.0/clients.conf > /dev/null << 'EOF'

client openvpn_server {
    ipaddr    = 10.10.10.254
    secret    = Passw0rd!
    shortname = ovpn
}
EOF

# Restart FreeRADIUS to apply pipeline and client configurations
systemctl restart freeradius
```

**Command Breakdown & Explanation:**

- `client openvpn_server`: Declares a unique client configuration block for the OpenVPN gateway.
- `ipaddr = 10.10.10.254`: Restricts accepted RADIUS traffic for this client entry to the specified gateway IP address.
- `secret = Passw0rd!`: Configures the shared secret password used to encrypt packet payloads between OpenVPN and FreeRADIUS.
- `systemctl restart freeradius`: Reloads all virtual server, client, and module definitions into memory.

## 4. [OpenVPN setup](/linux/vpn/ovpn-rad-ldap.md)

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate FreeRADIUS daemon state, port binding status, and client authorization rules on Debian 13 using standard terminal commands.

### 5.1 Verify FreeRADIUS service status

**Command:** `systemctl status freeradius`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/freeradius.service; enabled)`

### 5.2 Verify network port bindings for RADIUS authentication and accounting

**Command:** `ss -tuln | grep -E "1812|1813"`

**What it checks and variables to look for:**

- **State**: Must be `UNCONN`
- **Local Address:Port**: Must list `*:1812` and `*:1813`

### 5.3 Verify local LDAP authentication using radtest

**Command:** `radtest testuser "UserPassword" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Accept`
- **Code**: Must be `2`

### 5.4 Verify syntax and client configuration integrity

**Command:** `freeradius -C`

**What it checks and variables to look for:**

- **Configuration check**: Must return `Configuration appears to be OK`

<!-- Created by: Gergő Téringer, 2026 -->