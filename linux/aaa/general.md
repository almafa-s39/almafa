<!-- 
---
title: "General FreeRADIUS"
author: "Gergő Téringer"
---
 -->
# General FreeRADIUS

This document provides administrative procedures for installing, configuring, and testing a baseline FreeRADIUS deployment on Debian 13 (Trixie). It covers client definition, user authentication entries, debugging daemon execution, and verification steps using native utilities.

> [!NOTE]
> FreeRADIUS is the industry-standard modular Remote Authentication Dial-In User Service (RADIUS) server. On Debian 13, configuration files are structured under `/etc/freeradius/3.0/`.

## 1. FreeRADIUS Package Installation and Service Initializing

> [!IMPORTANT]
> Installing `freeradius-utils` alongside `freeradius` is required to obtain client diagnostic tools such as `radtest` for validating authentication requests locally.

```bash
# Update Debian package repositories and install FreeRADIUS core packages
apt install freeradius freeradius-utils

# Enable and start the FreeRADIUS daemon
systemctl enable freeradius --now
```

**Command Breakdown & Explanation:**

- `apt install freeradius freeradius-utils`: Updates repository metadata and installs the FreeRADIUS daemon along with diagnostic testing tools (`radtest`).
- `systemctl enable freeradius --now`: Configures the FreeRADIUS service to start automatically at system boot and immediately starts the service.

## 2. Network Client Configuration

Network Access Servers (NAS)—such as Wi-Fi Access Points, VPN gateways, or switches—must be explicitly authorized in FreeRADIUS with a IP address/subnet and shared secret before the server will accept authentication requests from them.

> [!WARNING]
> The shared secret defined in `clients.conf` must match the secret configured on the NAS device exactly. Using weak shared secrets leaves authentication traffic vulnerable to offline brute-force attacks.

```Bash
# Append a new Network Access Server (NAS) entry to clients.conf
tee -a /etc/freeradius/3.0/clients.conf > /dev/null << 'EOF'

client internal_network {
    ipaddr      = 192.168.10.0/24
    secret      = SuperSecretKey123!
    shortname   = internal-nas
}
EOF
```

**Command Breakdown & Explanation:**

- `client internal_network`: Defines a named configuration block for the client or network segment.
- `ipaddr = 192.168.10.0/24`: Specifies the single IP address or CIDR subnet permitted to send RADIUS requests to this server.
- `secret = SuperSecretKey123!`: Sets the shared encryption key used to obfuscate RADIUS attribute payloads (such as passwords).
- `shortname = internal-nas`: Provides a friendly alias used in server log entries.

## 3. User Credentials Configuration

Local user authentication records can be defined within the files module configuration. On Debian 13, user authorization entries are defined inside `/etc/freeradius/3.0/mods-config/files/authorize` (or symlinked via `/etc/freeradius/3.0/users`).

> [!TIP]
> FreeRADIUS supports various password attribute types. `Cleartext-Password` is used for basic PAP/CHAP validation, while production environments typically integrate with LDAP or Active Directory via MS-CHAPv2.

```Bash
# Define a local test user entry in the authorize file
tee -a /etc/freeradius/3.0/mods-config/files/authorize > /dev/null << 'EOF'

testuser Cleartext-Password := "UserPassword123!"
    Reply-Message = "Welcome to the Network!"
EOF
```

**Command Breakdown & Explanation:**

- `testuser`: The login identifier supplied in the Access-Request frame.
- `Cleartext-Password := "UserPassword123!"`: The comparison operator (`:=`) sets the required password string for user validation.
- `Reply-Message`: An optional RADIUS attribute sent back to the client in the `Access-Accept` response frame.

## 4. Running FreeRADIUS in Debug Mode

When troubleshooting authentication failures or policy evaluation issues, stop the systemd daemon and execute FreeRADIUS directly in foreground debug mode.

> [!CAUTION]
> FreeRADIUS cannot run in debug mode (`freeradius -X`) while the systemd `freeradius` service is running, because both instances attempt to bind to UDP ports `1812` and `1813`.

```Bash
# Stop the background systemd service
systemctl stop freeradius

# Launch FreeRADIUS in foreground debug mode
freeradius -X
```

**Command Breakdown & Explanation:**

- `systemctl stop freeradius`: Halts the background service to free UDP ports 1812 and 1813.
- `freeradius -X`: Launches FreeRADIUS in single-threaded foreground mode with verbose output enabled, detailing module loading, request processing, and credential matching in real time.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate service status, network port bindings, and local authentication processing on Debian 13 using standard system commands.

### 5.1 Verify FreeRADIUS service running status

**Command:** `systemctl status freeradius`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/freeradius.service; enabled)`

### 5.2 Verify UDP port listening state for authentication and accounting

**Command:** `ss -tuln | grep -E "1812|1813"`

**What it checks and variables to look for:**

- **State**: Must be `UNCONN` (UDP)
- **Local Address:Port**: Must display `*:1812` (Authentication) and `*:1813` (Accounting)

### 5.3 Verify local authentication processing using radtest

**Command:** `radtest testuser "UserPassword123!" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Accept`
- **Code**: Must be `2` (indicates successful Access-Accept packet type)
- **Reply-Message**: Must display `Welcome to the Network!`

### 5.4 Verify access rejection for invalid credentials

**Command:** `radtest testuser "WrongPassword" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Reject`
- **Code**: Must be `3` (indicates Access-Reject packet type)

<!-- Created by: Gergő Téringer, 2026 -->