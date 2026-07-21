<!-- 
---
title: "Kea DHCP"
author: "Gergő Téringer"
---
 -->
# Kea DHCP

This document provides administrative procedures for installing, configuring, and testing the Kea DHCP4 server alongside the Kea DHCP-DDNS module on Debian 13 (Trixie). It details environment preparation, Dynamic DNS integration using TSIG keys, and High Availability (Failover) configuration using Kea's hook libraries.

> [!NOTE]
> Kea uses JSON-formatted configuration files. Valid JSON syntax (including proper comma placement) is critical, as a single syntax error will prevent the daemon from starting.

## 1. Package Installation and File Preparation

The Kea suite is modular. You must install the DHCP4 server and the DDNS update module separately. After installation, stripping out the default comments creates a much cleaner configuration file to work with.

> [!IMPORTANT]
> The Kea daemon on Debian operates under the `_kea` user context. Ensure proper ownership is applied when generating backup files or new keys.

```bash
# Install the Kea DHCP4 and DDNS server packages from the attached ISO
apt install kea-dhcp4-server kea-dhcp-ddns-server

# Navigate to the configuration directory and create backups
cd /etc/kea
cp kea-dhcp4-server.conf kea4.bak
cp kea-dhcp-ddns-server.conf kea-ddns.bak

# Adjust ownership to ensure the Kea service can access the files
chown _kea:root ./*

# Strip out commented lines to create clean configuration files
grep -v "//" kea4.bak > kea-dhcp4-server.conf
grep -v "//" kea-ddns.bak > kea-dhcp-ddns-server.conf
```

**Command Breakdown & Explanation:**

- `apt install kea-dhcp4-server kea-dhcp-ddns-server`: Installs the core IPv4 DHCP engine and the Dynamic DNS helper module.
- `cp ...`: Creates `.bak` copies of the massive default configuration files before stripping them.
- `chown _kea:root ./*`: Ensures the `_kea` system user maintains read/write permissions for all files in the directory.
- `grep -v "//"`: Filters out any line containing standard JSON/C-style `//` comments, redirecting the clean output to the production configuration files.

## 2. IPv4 DHCP and DDNS Integration

To instruct the Kea DHCP4 server to send DNS update requests to the Kea DDNS daemon, you must embed the DDNS parameters within the main `Dhcp4` JSON structure of `/etc/kea/kea-dhcp4-server.conf`.

```json
"dhcp-ddns": { "enable-updates": true },
"ddns-qualifying-suffix": "unitel.com",
"ddns-override-client-update": true,
```

**Command Breakdown & Explanation:**

- `dhcp-ddns: { "enable-updates": true }`: Globally enables the DHCP server to generate DDNS update events.
- `ddns-qualifying-suffix`: Specifies the default domain suffix appended to client hostnames (e.g., `client1.unitel.com`).
- `ddns-override-client-update`: Forces the server to handle DNS updates directly, even if the client requests to handle its own updates via DHCP options.

## 3. Dynamic DNS (DDNS) Server Configuration

The Kea DDNS daemon (`kea-dhcp-ddns-server`) communicates securely with BIND9 (or other DNS servers) using TSIG keys. You must generate a key, format it as JSON, and import it into the DDNS configuration.

> [!CAUTION]
> The TSIG key file (`/etc/kea/ddns.json`) contains the raw base64 secret. It must be readable by the `_kea` user but secured from general access.

```bash
# Generate a TSIG key (requires the bind9 or bind9utils package installed)
tsig-keygen "ddns" > /etc/kea/ddns.key

# Copy the file to prepare it for JSON formatting
cp /etc/kea/ddns.key /etc/kea/ddns.json
```

Modify `/etc/kea/ddns.json` to match the exact JSON array format below.

> [!IMPORTANT]
> Do not forget the trailing comma (`,`) at the end of the array closure, as this block will be injected directly into the larger DDNS configuration structure!

```json
"tsig-keys": [{
    "name": "<name>",
    "algorithm": "hmac-sha256",
    "secret": "<BASE64>"   
}],
```

Next, edit `/etc/kea/kea-dhcp-ddns-server.conf`. Remove the default `tsig-keys` block, include the external JSON file using the `<?include?>` macro, and define your forward and reverse zones.

```json
<?include "/etc/kea/ddns.json" ?>
"forward-ddns": {
    "ddns-domains": [{
        "name": "<domain>.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNSv4_address>" }]
    }]
},

"reverse-ddns": {
    "ddns-domains": [{
        "name": "<reverse-domain>.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNSv4_address>" }]
    }]
},
```

**Command Breakdown & Explanation:**

- `<?include "/etc/kea/ddns.json" ?>`: A Kea-specific macro that dynamically injects the contents of the TSIG key file into the running configuration, keeping secrets isolated.
- `forward-ddns`: Maps the domain zone (`<domain>.`) to its authoritative DNS server (`<DNSv4_address>`) and the required TSIG key.
- `reverse-ddns`: Maps the reverse lookup zone (e.g., `2.20.172.in-addr.arpa.`) for PTR record updates.

## 4. High Availability (Failover) Configuration

Kea implements High Availability (HA) through hook libraries rather than native core directives. You must load the HA library in `/etc/kea/kea-dhcp4-server.conf` and define the peer topology.

> [!TIP]
> The `libdhcp_lease_cmds.so` library is often required alongside `libdhcp_ha.so` to allow the HA module to fetch and sync lease states via the control API.

```json
{
"Dhcp4": {
    "hooks-libraries": [
        {
            "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_lease_cmds.so"
        },
        {
            "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_ha.so",
            "parameters": {
                "high-availability": [{
                    "this-server-name": "srv1",
                    "mode": "hot-standby",
                    "peers": [
                        {
                            "name": "srv1",
                            "url": "http://172.20.2.1:9999/",
                            "role": "primary"
                        },
                        {
                            "name": "srv2",
                            "url": "http://172.20.2.2:9999/",
                            "role": "standby"
                        }
                    ]
                }]
            }
        }
    ]
    // Remaining DHCP4 configuration follows here...
}
}
```

**Command Breakdown & Explanation:**

- `hooks-libraries`: An array defining external shared object (`.so`) libraries that extend Kea's functionality.
- `this-server-name`: Instructs the local daemon which peer profile it represents (must match exactly one `name` in the `peers` array).
- `mode`: `hot-standby` establishes an Active/Passive relationship.
- `url`: The HTTP endpoint and port where the Kea API listens for peer synchronization traffic.
- `role`: Defines `primary` and `standby` designations for the heartbeat and lease synchronization logic.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the Kea service statuses, configuration syntax, and network port bindings on Debian 13 using standard administrative commands.

### 5.1 Verify Kea DHCP4 configuration syntax

**Command:** `kea-dhcp4 -t /etc/kea/kea-dhcp4-server.conf`

**What it checks and variables to look for:**

- **Syntax output**: Must output `Configuration file OK` (If JSON parsing fails, it will specify the exact line and character column of the error).

### 5.2 Verify Kea DDNS configuration syntax

**Command:** `kea-dhcp-ddns -t /etc/kea/kea-dhcp-ddns-server.conf`

**What it checks and variables to look for:**

- **Syntax output**: Must output `Configuration file OK`.

### 5.3 Verify Kea DHCP4 service status

**Command:** `systemctl status kea-dhcp4-server`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/kea-dhcp4-server.service; enabled)`

### 5.4 Verify Kea DDNS service status

**Command:** `systemctl status kea-dhcp-ddns-server`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/kea-dhcp-ddns-server.service; enabled)`

### 5.5 Verify network listening state on DHCP and HA ports

**Command:** `ss -tulnp | grep -E "67|9999"`

**What it checks and variables to look for:**

- **State (Port 67)**: Must be `UNCONN` bound to `*:67` or `0.0.0.0:67` (DHCP Service)
- **State (Port 9999)**: Must be `LISTEN` bound to the designated HA IP (e.g., `172.20.2.1:9999`)

<!-- Created by: Gergő Téringer, 2026 -->