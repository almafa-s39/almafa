<!-- 
---
title: "OpenVPN with RADIUS Authentication Configuration Guide"
author: "Gergő Téringer"
---
 -->
# OpenVPN with RADIUS Authentication Configuration Guide

This document provides administrative procedures for configuring an OpenVPN server to authenticate remote clients against a RADIUS server using the Pluggable Authentication Modules (PAM) framework. It includes configuration breakdowns and verification test cases.

> [!NOTE]
> OpenVPN does not speak the RADIUS protocol natively. Instead, it uses a PAM plugin to pass client credentials to the host OS, which then uses `libpam-radius-auth` to forward the request to the backend RADIUS server.

## 1. Package Installation

Install the OpenVPN daemon and the PAM RADIUS authentication library.

```Bash
# Install the required packages
apt install openvpn libpam-radius-auth
```

**Command Breakdown & Explanation:**

- `openvpn`: The core VPN server and client daemon.
- `libpam-radius-auth`: A PAM module that allows any PAM-aware application (like OpenVPN) to authenticate users against a remote RADIUS server.

## 2. OpenVPN Server Configuration

Edit your OpenVPN server profile (typically located at `/etc/openvpn/server/server.conf` or similar) to load the PAM authentication plugin and configure it to use the `openvpn` PAM service.

```Plaintext
# Append the following lines to your OpenVPN server configuration file
setenv deferred_auth_pam 1
plugin /usr/lib/openvpn/openvpn-plugin-auth-pam.so "openvpn login USERNAME password PASSWORD"
```

**Configuration Breakdown & Explanation:**

- `setenv deferred_auth_pam 1`: Enables asynchronous PAM authentication, preventing the main OpenVPN process from blocking while waiting for the RADIUS server to respond.
- `plugin ... openvpn-plugin-auth-pam.so`: Loads the PAM plugin.
- `"openvpn login USERNAME password PASSWORD"`: Instructs the plugin to use the PAM service file named `openvpn` (which we will create next) and maps the username and password fields.

## 3. PAM Service Configuration (/etc/pam.d/openvpn)

Create the dedicated PAM service file for OpenVPN. This file dictates that authentication should be handled by the RADIUS module, while account and session management are universally permitted.

```Bash
# Create and edit the PAM openvpn service file
cat << 'EOF' > /etc/pam.d/openvpn
auth sufficient pam_radius_auth.so
account sufficient pam_permit.so
session sufficient pam_permit.so
EOF
```

**Configuration Breakdown & Explanation:**

- `auth sufficient pam_radius_auth.so`: Tells PAM to send authentication requests to the RADIUS server configured in the system. If it succeeds, authentication is immediately granted (`sufficient`).
- `account/session sufficient pam_permit.so`: Bypasses strict local account and session tracking (like checking `/etc/shadow` expiration or creating home directories), which are unnecessary for VPN-only users.

## 4. RADIUS Server Connection (/etc/pam_radius_auth.conf)

Configure the PAM RADIUS module with the IP address, shared secret, and timeout values of your RADIUS server.

```Plaintext
# Edit /etc/pam_radius_auth.conf
# Comment out any unused server options, and configure only the used ones
# Format: server[:port] shared_secret timeout (s) source_ip vrf

127.0.0.1 Skill39$$ 60
```

**Configuration Breakdown & Explanation:**

- `127.0.0.1`: The IP address of the RADIUS server (in this example, it is running on the local host).
- `Skill39$$`: The shared secret key used to encrypt the RADIUS payload between the PAM module and the RADIUS server.
- `60`: The timeout in seconds to wait for a response from the RADIUS server.

## 5. OpenVPN Client Configuration

The client profile must be instructed to prompt the user for credentials and send them to the server.

```Plaintext
# Edit your OpenVPN client profile (.ovpn), and add the following line
auth-user-pass
```

**Configuration Breakdown & Explanation:**

- `auth-user-pass`: Instructs the OpenVPN client software to prompt the user for a username and password and forward them securely to the server during the TLS handshake.

*Restart the OpenVPN server service to apply the plugin and PAM configurations.*

```Bash
systemctl restart openvpn-server@server
```

## 6. Verification and Test Cases

> [!NOTE]
> Validate the OpenVPN server status, test the RADIUS connection locally, and monitor the live logs during a client connection attempt.

### 6.1 Verify OpenVPN Service Status

**Command:** `systemctl status openvpn-server@server` *(adjust `@server` to match your config file name)*

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`.
- **Logs**: Look for `PLUGIN_INIT: POST` and `PLUGIN_INIT: plugin initialization function succeeded` to confirm the PAM module loaded without path errors.

### 6.2 Test Case: Verify Client Authentication (Live Logs)

**Command:** `tail -f /var/log/syslog` *(or `/var/log/messages` depending on your OS logging configuration)*

**Action:** Attempt to connect using the OpenVPN client with valid RADIUS credentials.

**What it checks and variables to look for:**

- **RADIUS Handshake**: The syslog will capture the PAM module's interaction. Look for `pam_radius_auth: Got RADIUS response code 2` (Access-Accept). Code 3 indicates Access-Reject.
- **OpenVPN Auth**: Following the PAM success, the OpenVPN logs should display `TLS: Username/Password authentication succeeded for username`.

<!-- Created by: Gergő Téringer, 2026 -->