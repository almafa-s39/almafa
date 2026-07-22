<!-- 
---
title: "SNMP Client and Server Configuration"
author: "Gergő Téringer"
---
 -->
# SNMP Client and Server Configuration

This guide details the setup of the SNMP (Simple Network Management Protocol) agent and client utilities on Debian 13 Trixie. It covers legacy SNMPv1 and SNMPv2c configurations alongside the modern, secure SNMPv3 standard, and demonstrates how to test the configurations using the `snmpwalk` utility.

> [!NOTE]
> SNMPv1 and SNMPv2c transmit data, including the community string (password), in cleartext over the network. In modern environments involving Windows Server 2025 or Windows 11 clients, strict security policies often deprecate or block plaintext SNMP traffic by default. It is highly recommended to use SNMPv3 with `authPriv` (Authentication and Privacy) for production networks.

## 1. Package Installation

Install the SNMP daemon (server/agent) and the SNMP client utilities, which provide the `snmpwalk` command used for polling and testing.

```Bash
apt install snmpd snmp libsnmp-dev
systemctl enable snmpd --now
```

**Command Breakdown & Explanation:**

- `apt install snmpd snmp libsnmp-dev`: Installs the SNMP agent (`snmpd`) to serve data, the client tools (`snmp`) to query data, and development libraries for extensive MIB (Management Information Base) support.
- `systemctl enable snmpd --now`: Enables the SNMP daemon to start at system boot and launches it immediately.

## 2. Server Configuration (snmpd)

The primary configuration file for the SNMP agent is `/etc/snmp/snmpd.conf`. Before making changes, it is good practice to back up the original configuration to preserve the default examples.

```Bash
mv /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.orig
nano /etc/snmp/snmpd.conf
```

**Command Breakdown & Explanation:**

- `mv`: Renames the default configuration file, moving it out of the way to serve as a backup.
- `nano`: Opens a fresh, blank configuration file to define our custom SNMP settings cleanly.

### 2.1 Global Server Settings

First, define the system location, contact information, and the listening addresses for the server. This provides basic administrative context when the server is queried.

```Bash
# Add to /etc/snmp/snmpd.conf
sysLocation ServerRoom-Rack1
sysContact Admin <admin@company.com>

# Listen on all IPv4 and IPv6 interfaces on standard UDP port 161
agentAddress udp:161,udp6:[::1]:161
```

### 2.2 SNMPv1 and SNMPv2c Configuration

> [!WARNING]
> Use strong, unpredictable community strings if you must rely on v1/v2c for legacy hardware integration. Never use the default `public` or `private` strings in a live production environment.

```Bash
# Add to /etc/snmp/snmpd.conf

# Define a read-only community string for SNMPv1/v2c
rocommunity secureCommString default
```

### 2.3 SNMPv3 Configuration

SNMPv3 introduces robust security through cryptographic authentication and payload encryption.

> [!IMPORTANT]
> Unlike v1/v2c, SNMPv3 users must be created while the `snmpd` service is completely stopped (or injected via the `net-snmp-config` utility). We will use the persistent state file method here.

```Bash
# Stop the service before adding the v3 user
systemctl stop snmpd

# Add the v3 user to the persistent state file (NOT the main config)
nano /var/lib/snmp/snmpd.conf
```

```Bash
# Append this line to /var/lib/snmp/snmpd.conf
createUser myv3user SHA "MyAuthPass123" AES "MyPrivPass123"
```

Next, grant this newly created user access permissions inside the main configuration file.

```Bash
nano /etc/snmp/snmpd.conf
```

```Bash
# Add this line to /etc/snmp/snmpd.conf
rouser myv3user authpriv
```

Finally, start the SNMP service. The daemon will securely parse the user from the state file and apply the access rules.

```Bash
systemctl start snmpd
```

**Command Breakdown & Explanation:**

- `systemctl stop snmpd`: The daemon automatically overwrites `/var/lib/snmp/snmpd.conf` on exit. You must stop it *before* making manual edits to prevent your credentials from being erased.
- `createUser`: Directive that creates a v3 user named `myv3user`, using `SHA` for authentication hashing and `AES` for payload encryption.
- `rouser`: Grants read-only access to `myv3user` requiring `authpriv` (both authentication and privacy/encryption must be successfully negotiated in the client request).

## 3. Client Configuration and Testing (snmpwalk)

The `snmpwalk` tool systematically queries a network entity for a tree of information. It acts as the client, fetching data from the server configuration we just deployed.

### 3.1 Testing SNMPv1 and SNMPv2c

> [!TIP]
> Most modern client queries default to SNMPv2c instead of v1 due to its support for `GetBulk` requests, which retrieve large amounts of management data efficiently in fewer packets.

```Bash
# Testing SNMPv2c
snmpwalk -v 2c -c secureCommString 127.0.0.1 system

# Testing SNMPv1 (Legacy)
snmpwalk -v 1 -c secureCommString 127.0.0.1 system
```

**Command Breakdown & Explanation:**

- `-v 2c`: Specifies the SNMP protocol version to use (2c).
- `-c secureCommString`: Provides the community string (password) configured in the server settings.
- `127.0.0.1`: The target IP address of the SNMP agent.
- `system`: The specific OID (Object Identifier) tree to walk (retrieving basic system information like uptime, contact, and hostname).

### 3.2 Testing SNMPv3

SNMPv3 client queries are significantly more verbose because all authentication and encryption parameters must be explicitly declared in the command line syntax.

```Bash
snmpwalk -v 3 -l authPriv -u myv3user -a SHA -A "MyAuthPass123" -x AES -X "MyPrivPass123" 127.0.0.1 system
```

**Command Breakdown & Explanation:**

- `-v 3`: Specifies SNMP version 3.
- `-l authPriv`: Sets the required security level to Authentication and Privacy (encryption).
- `-u myv3user`: Specifies the security name (username).
- `-a SHA`: Specifies the authentication protocol hash.
- `-A "MyAuthPass123"`: Provides the authentication password.
- `-x AES`: Specifies the privacy (encryption) protocol.
- `-X "MyPrivPass123"`: Provides the privacy password to decrypt the payload.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the service statuses and UDP network port bindings on Debian 13 to ensure the SNMP agent is actively listening for client requests and hasn't crashed due to syntax errors in the configuration file.

### 4.1 Verify snmpd status

**Command:** `systemctl status snmpd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/snmpd.service; enabled;...)`

### 4.2 Verify network listening state on port 161

**Command:** `ss -uln | grep :161`

**What it checks and variables to look for:**

- **State**: Must be `UNCONN` (Because SNMP uses UDP, which is a connectionless protocol, `UNCONN` signifies an open and listening port).
- **Local Address:Port**: Must display `*:161`, `0.0.0.0:161`, or `[::]:161`

<!-- Created by: Gergő Téringer, 2026 -->