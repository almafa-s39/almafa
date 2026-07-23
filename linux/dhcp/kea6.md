<!-- 
---
title: "Kea DHCP6 Server"
author: "Gergő Téringer"
---
 -->
# Kea DHCP6 Server

This document provides administrative procedures for installing,
configuring, and testing the Kea DHCPv6 server alongside the Kea
DHCP-DDNS module on Debian 13 (Trixie). It details environment
preparation, Dynamic DNS integration using TSIG keys, and High
Availability (Failover) configuration using Kea's hook libraries.

> [!NOTE]
> Kea uses JSON-formatted configuration files. Valid JSON syntax
> (including proper comma placement) is critical, as a single syntax
> error will prevent the daemon from starting.

## 1. Package Installation and File Preparation

The Kea suite is modular. You must install the DHCPv6 server package and
the DDNS update module package separately. After installation, stripping
out the default comments creates a much cleaner configuration file to
work with.

> [!IMPORTANT]
> The Kea daemon on Debian operates under the `_kea` user context. Ensure
> proper ownership is applied when generating backup files or new keys.

```bash
# Install the Kea DHCPv6 server and the DHCP-DDNS (D2) update module
apt install kea-dhcp6-server kea-dhcp-ddns-server

# Navigate to the configuration directory and create backups
cd /etc/kea
cp kea-dhcp6-server.conf kea6.bak
cp kea-dhcp-ddns-server.conf kea-ddns.bak

# Adjust ownership to ensure the Kea service can access the files
chown _kea:root ./*

# Strip out commented lines to create clean configuration files
grep -v "//" kea6.bak > kea-dhcp6-server.conf
grep -v "//" kea-ddns.bak > kea-dhcp-ddns-server.conf
```

**Command Breakdown & Explanation:**

- `apt install kea-dhcp6-server kea-dhcp-ddns-server`: Installs the core
  DHCPv6 engine and the Dynamic DNS (D2) helper module as two separate
  packages, both of which are required for this setup.
- `cp ...`: Creates `.bak` copies of the default configuration files
  before stripping them, for both the DHCPv6 server and the DDNS daemon.
- `chown _kea:root ./*`: Ensures the `_kea` system user maintains
  read/write permissions for all files in the directory.
- `grep -v "//"`: Filters out any line containing standard JSON/C-style
  `//` comments, redirecting the clean output to the production
  configuration files.

## 2. IPv6 DHCP and DDNS Integration

To instruct the Kea DHCPv6 server to send DNS update requests to the Kea
DDNS daemon, you must embed the DDNS parameters within the main `Dhcp6`
JSON structure of `/etc/kea/kea-dhcp6-server.conf`. For IPv6 clients,
these updates create `AAAA` records (and corresponding `PTR` records in
the reverse zone) rather than the `A` records used for IPv4.

```json
"dhcp-ddns": { "enable-updates": true },
"ddns-qualifying-suffix": "unitel.com",
"ddns-override-client-update": true,
```

**Command Breakdown & Explanation:**

- `dhcp-ddns: { "enable-updates": true }`: Globally enables the DHCPv6
  server to generate DDNS update events.
- `ddns-qualifying-suffix`: Specifies the default domain suffix appended
  to client hostnames (e.g., `client1.unitel.com`).
- `ddns-override-client-update`: Forces the server to handle DNS updates
  directly, even if the client requests to handle its own updates via
  DHCPv6 options.

> [!NOTE]
> These three parameters sit inside the top-level `Dhcp6` object (see
> Section 4 for the full structure), not as a standalone file.

## 3. Dynamic DNS (DDNS) Server Configuration

The Kea DDNS daemon (`kea-dhcp-ddns-server`) communicates securely with
BIND9 (or other DNS servers) using TSIG keys. You must generate a key,
format it as JSON, and import it into the DDNS configuration.

> [!CAUTION]
> The TSIG key file (`/etc/kea/ddns.json`) contains the raw base64
> secret. It must be readable by the `_kea` user but secured from
> general access.

```bash
# Generate a TSIG key (requires the bind9 or bind9utils package installed)
tsig-keygen "ddns" > /etc/kea/ddns.key

# Copy the file to prepare it for JSON formatting
cp /etc/kea/ddns.key /etc/kea/ddns.json
```

Modify `/etc/kea/ddns.json` to match the exact JSON array format below.

> [!IMPORTANT]
> Do not forget the trailing comma (`,`) at the end of the array closure,
> as this block will be injected directly into the larger DDNS
> configuration structure!

```json
"tsig-keys": [{
    "name": "<name>",
    "algorithm": "hmac-sha256",
    "secret": "<BASE64>"
}],
```

Next, edit `/etc/kea/kea-dhcp-ddns-server.conf`. Remove the default
`tsig-keys` block, include the external JSON file using the `<?include?>`
macro, and define your forward and reverse zones. The reverse zone for
IPv6 uses the `ip6.arpa` domain in nibble format (each hex digit of the
fully expanded address, reversed and dot-separated) rather than the
`in-addr.arpa` format used for IPv4.

```json
<?include "/etc/kea/ddns.json" ?>
"forward-ddns": {
    "ddns-domains": [{
        "name": "<domain>.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNS_server_address>" }]
    }]
},

"reverse-ddns": {
    "ddns-domains": [{
        "name": "<reverse-nibble-domain>.ip6.arpa.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNS_server_address>" }]
    }]
},
```

**Command Breakdown & Explanation:**

- `<?include "/etc/kea/ddns.json" ?>`: A Kea-specific macro that
  dynamically injects the contents of the TSIG key file into the running
  configuration, keeping secrets isolated.
- `forward-ddns`: Maps the domain zone (`<domain>.`) to its authoritative
  DNS server (`<DNS_server_address>`) and the required TSIG key. This
  zone will receive `AAAA` record updates for IPv6 client addresses.
- `reverse-ddns`: Maps the IPv6 reverse lookup zone
  (`<reverse-nibble-domain>.ip6.arpa.`) for `PTR` record updates. For
  example, the reverse zone for prefix `2001:db8:1::/48` is expressed as
  `1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.` (the /48 boundary reversed,
  nibble by nibble).

> [!TIP]
> IPv6 reverse zones are long and error-prone to type by hand. Use a
> nibble-boundary calculator or `dig -x` against a known address to
> confirm the exact zone name before committing it to the configuration.
> [!NOTE]
> `<DNS_server_address>` is the address of the authoritative DNS server
> Kea sends updates to. This is independent of the DHCP protocol version
> in use: the target DNS server can be reached over IPv4 or IPv6
> transport regardless of whether the DHCP leases themselves are IPv6.
> Use whichever address family the DNS server actually listens on.

## 4. High Availability (Failover) Configuration

Kea implements High Availability (HA) through hook libraries rather than
native core directives. You must load the HA library in
`/etc/kea/kea-dhcp6-server.conf` and define the peer topology.

> [!TIP]
> The `libdhcp_lease_cmds.so` library is often required alongside
> `libdhcp_ha.so` to allow the HA module to fetch and sync lease states
> via the control API.

```json
{
"Dhcp6": {
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
                            "url": "http://[2001:db8:2::1]:9999/",
                            "role": "primary"
                        },
                        {
                            "name": "srv2",
                            "url": "http://[2001:db8:2::2]:9999/",
                            "role": "standby"
                        }
                    ]
                }]
            }
        }
    ]
    // Remaining Dhcp6 configuration follows here...
}
}
```

**Command Breakdown & Explanation:**

- `Dhcp6`: The top-level scope for the DHCPv6 server configuration. Kea
  requires this key capitalized exactly as `Dhcp6`; a lowercase `dhcp6`
  key will fail to parse.
- `hooks-libraries`: An array defining external shared object (`.so`)
  libraries that extend Kea's functionality.
- `this-server-name`: Instructs the local daemon which peer profile it
  represents (must match exactly one `name` in the `peers` array).
- `mode`: `hot-standby` establishes an Active/Passive relationship.
- `url`: The HTTP endpoint and port where the Kea Control Agent API
  listens for peer synchronization traffic. IPv6 literal addresses must
  be enclosed in square brackets (`[2001:db8:2::1]`) in the URL, per
  standard URI syntax.
- `role`: Defines `primary` and `standby` designations for the heartbeat
  and lease synchronization logic.

> [!WARNING]
> The HA control channel itself (the `url` values above) can run over
> IPv4 or IPv6 independently of the DHCPv6 lease traffic being managed.
> If your management network is IPv4-only, use IPv4 addresses for the
> `url` fields instead — this does not need to match the address family
> of the leases being replicated.

## 5. Verification and Troubleshooting

Validate the Kea service statuses, configuration syntax, and network
port bindings on Debian 13 using standard administrative commands.

### 5.1 Verify Kea DHCPv6 configuration syntax

kea-dhcp6 -t /etc/kea/kea-dhcp6-server.conf

What it checks and variables to look for:

- **Syntax output**: Must output `Configuration file OK` (If JSON
  parsing fails, it will specify the exact line and character column of
  the error).

### 5.2 Verify Kea DDNS configuration syntax

```bash
kea-dhcp-ddns -t /etc/kea/kea-dhcp-ddns-server.conf
```

What it checks and variables to look for:

- **Syntax output**: Must output `Configuration file OK`.

### 5.3 Verify Kea DHCPv6 service status

```bash
systemctl status kea-dhcp6-server
```

What it checks and variables to look for:

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/kea-dhcp6-server.service; enabled)`

### 5.4 Verify Kea DDNS service status

```bash
systemctl status kea-dhcp-ddns-server
```

What it checks and variables to look for:

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/kea-dhcp-ddns-server.service; enabled)`

### 5.5 Verify network listening state on DHCPv6 and HA ports

```bash
ss -tulnp | grep -E "547|9999"
```

What it checks and variables to look for:

- **State (Port 547)**: Must be `UNCONN` bound to `*:547` or `:::547`
  (DHCPv6 server port; clients send to UDP `546`, the server listens on
  UDP `547`).
- **State (Port 9999)**: Must be `LISTEN` bound to the designated HA
  address (e.g., `[2001:db8:2::1]:9999` or `172.20.2.1:9999`, depending
  on which address family the HA control channel uses).

> [!NOTE]
> Unlike DHCPv4 (UDP port `67`), DHCPv6 uses UDP port `547` for
> server-bound traffic. If you previously used this playbook for a
> DHCPv4 deployment, double-check that every port reference has been
> updated — a leftover `67` will cause this verification step to
> silently report nothing.

<!-- Created by: Gergő Téringer, 2026 -->