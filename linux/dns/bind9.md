<!-- 
---
title: "Bind9 DNS Server Configuration"
author: "Gergő Téringer"
---
 -->
# Bind9 DNS Server Configuration

This guide details the installation and configuration of the Bind9 DNS server on Debian 13 Trixie. It covers primary, secondary, and forwarding configurations, along with essential DNS record types and AppArmor permissions.

> [!NOTE]
> Ensure that your server has a static IP address configured before setting up Bind9. Modern OS environments (like Windows 11 or Windows Server 2025) aggressively cache DNS and will fail to resolve hostnames consistently if the core DNS server IP changes dynamically.

## 1. Packages

Install the core Bind9 daemon, essential utilities for management and troubleshooting, and the documentation package for offline reference.

```Bash
# Install the Bind9 packages
apt install bind9 bind9-utils bind9-doc
systemctl enable bind9 --now
```

**Command Breakdown & Explanation:**

- `apt install bind9 bind9-utils bind9-doc`: Installs the Bind9 server, useful tools like `rndc` or `dig` (via utils), and local documentation.
- `systemctl enable bind9 --now`: Enables the service to start on boot and starts it immediately.

## 2. Configuration Options

### 2.1 Default options to include

These default options are mandatory for competition environments to ensure proper operation with minimal effort. In Debian 13 Trixie, these are no longer included by default in the `/etc/bind/named.conf.options` file and must be added manually.

```Bash
# /etc/bind/named.conf.options
options {
    # Enable recursion and allow it to acl.
    recursion yes;
    allow-recursion { any; };
    allow-query-cache { any; };
    
    dnssec-validation no;

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };
};
```

**Command Breakdown & Explanation:**

- `recursion yes;`: Enables recursive queries.
- `allow-recursion { any; };`: Defines which clients can make recursive queries.
- `allow-query-cache { any; };`: Allows clients to query the server's cache.
- `dnssec-validation no;`: Disables DNSSEC validation, which is crucial in closed environments to prevent resolution failures due to missing internet trust anchors.
- `forwarders`: Forwards unresolved queries to external public DNS servers.

### 2.2 Logging

Configure logging to simplify troubleshooting. This isolates separate channels for query logs and general daemon logs.

```Bash
# /etc/bind/named.conf.options or included logging file
logging {
    # You can create the channel queries, which will be where and how will you log.
    channel query {
        file "/var/lib/bind/query.log";
        print-time yes;
        print-category yes;
        print-severity yes;
        severity debug;
    };

    channel default {
        file "/var/lib/bind/all.log";
        print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
    };

    # You can create categories which will set what type of categories send to which channel.
    # Transfer logs
    category notify { default; };
    category xfer-in { default; };
    category xfer-out { default; };

    # DDNS
    category update { default; };
    category update-security { default; };

    # Queries
    category queries { query; };
    category query-errors { query; };

    # DNSSEC
    category dnssec { default; };

    # Default
    category default { default; };
};
```

**Command Breakdown & Explanation:**

- `channel`: Defines a logging destination and its format.
- `file`: Specifies the absolute path to the log file.
- `severity`: Sets the verbosity level (`debug` for detailed queries, `info` for general daemon activity).
- `category`: Maps internal Bind9 log categories to the defined channels.

## 3. Main Configuration File

The main configuration file is `/etc/bind/named.conf`. It is standard practice to include other configuration files from here rather than placing all configurations in a single file.

> [!WARNING]
> If you are using views, do not include `named.conf.root-hints` in the main file, or make sure to configure your views in that file as well! For custom server configurations, it is best to use `/etc/bind/named.conf.local`, or create a new file and include it.

## 4. Primary Server Configuration

This configuration demonstrates how to use views to separate lookups by source IP, configure zone transfers, and allow dynamic updates.

> [!TIP]
> If you use the exact same configuration for all zones without splitting them by views, place the configuration directly in the `options` directive of `/etc/bind/named.conf.options` to apply it globally.

```Bash
# To generate the following key, run this in your shell:
# tsig-keygen "update" > /etc/bind/update.key

# In your config file (e.g., /etc/bind/named.conf.local)
include "/etc/bind/update.key";

acl intra { 10.0.0.0/8; };
acl mydns { 
    192.168.100.101;
    192.168.100.102;
    192.168.100.103;
};

acl mydhcp { 
    10.0.0.1;
};

view inside {
    match-clients { intra; };
    zone company.com {
        type master;
        # Use /var/lib/bind/ for DDNS to avoid AppArmor issues, or /etc/bind/ for static files.
        file "/var/lib/bind/company.com.in"; 
        allow-transfer { "mydns"; };
        also-notify { "mydns"; };
        allow-update { "mydhcp"; key "update"; };
    };
};

view outside {
    match-clients { any; };
    zone company.com {
        type master;
        file "/var/lib/bind/company.com.ex"; 
        allow-transfer { "mydns"; };
        also-notify { "mydns"; };
    };
};
```

**Command Breakdown & Explanation:**

- `tsig-keygen`: Generates a secure TSIG key for Dynamic DNS (DDNS) updates.
- `acl`: Defines an Access Control List to logically group IP addresses.
- `view`: Isolates DNS responses based on the source IP (`match-clients`).
- `type master`: Designates this server as the authoritative primary source for the zone.
- `allow-update`: Permits specific clients (e.g., DHCP servers) to dynamically update DNS records using the provided TSIG key.

## 5. Secondary Server Configuration

The secondary (slave) server continuously synchronizes zone data from the primary server.

```Bash
# /etc/bind/named.conf.local
include "/etc/bind/update.key";

acl intra { 10.0.0.0/8; };
acl mydns { 
    192.168.100.100;
};

view inside {
    match-clients { intra; };
    zone company.com {
        type slave;
        file "/var/lib/bind/company.com.in"; 
        masters { "mydns"; };
    };
};

view outside {
    match-clients { any; };
    zone company.com {
        type slave;
        file "/var/lib/bind/company.com.ex"; 
        masters { "mydns"; };
    };
};
```

**Command Breakdown & Explanation:**

- `type slave`: Configures the zone to act as a secondary replica.
- `masters`: Specifies the IP address of the primary server from which to pull the zone transfers.

## 6. Forwarder Server Configuration (Conditional Forwarding)

Conditional forwarding directs queries for a specific domain to designated external DNS servers rather than attempting recursive resolution.

```Bash
zone google.com {
    type forward;
    forwarders {
        8.8.8.8;
    };
};
```

**Command Breakdown & Explanation:**

- `type forward`: Instructs Bind9 to only forward queries for this specific zone.
- `forwarders`: The target DNS servers designated to handle the resolution for the specified domain.

## 7. Reverse Zone Configuration

Reverse zones map IP addresses back to DNS names. Ensure you strictly follow the `.in-addr.arpa` (IPv4) and `.ip6.arpa` (IPv6) naming conventions.

```Bash
# For IPv4 Subnet: 10.20.30.0/24
zone 30.20.10.in-addr.arpa {
    type master;
    file "/var/lib/bind/db.10.20.30";
};

# For IPv6 Subnet: 2001:db8:1010:1010::/64
zone 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa {
    type master;
    file "/var/lib/bind/db.2001.db8";
};
```

**Command Breakdown & Explanation:**

- `in-addr.arpa`: Standard suffix for IPv4 reverse lookup zones, reversing the first three octets.
- `ip6.arpa`: Standard suffix for IPv6 reverse lookup zones, reversing every nibble of the prefix.

## 8. DNS Records

### 8.1 A Record

Maps a DNS name directly to an IPv4 address.

```Bash
# Syntax: <NAME>  A   <IPv4>
www     A   10.10.10.10
```

### 8.2 AAAA Record

Maps a DNS name directly to an IPv6 address.

```Bash
# Syntax: <NAME>  AAAA    <IPv6>
www     AAAA    2001:db8:1010::1010
```

### 8.3 CNAME Record

Maps an alias DNS name to a canonical DNS name.

> [!IMPORTANT]
> The target name **must** end with a trailing dot (`.`)!

```Bash
# Syntax: <NAME>    CNAME   <TARGET>
web       CNAME   www.domain.name.
```

### 8.4 NS Record

Delegates a DNS zone to an authoritative name server. You must configure a corresponding A or AAAA record for the target NS!

```Bash
# Syntax: <NAME>          NS <TARGET>
domain.name     NS ns1.domain.name.
```

### 8.5 SOA Record

Marks the beginning of a DNS zone and identifies its authoritative properties. This is **REQUIRED** for all zones!

- **MNAME**: Primary nameserver (DDNS server if applicable).
- **RNAME**: Admin contact email name (with the `@` replaced by a `.`).
- **Serial**: Zone version (Crucial for triggering zone transfers).
- **Refresh**: Delay between secondary server synchronization checks.
- **Retry**: Delay after a failed sync attempt.
- **Expire**: Time before a secondary marks its data as non-authoritative if unreachable.
- **Minimum**: Global negative caching TTL (Time-To-Live).

```Bash
# Syntax: @   SOA   <TARGET>.         <EMAIL>.                 ( <SERIAL> <REFRESH> <RETRY> <EXPIRE> <MINUMUM> )
@   SOA   ns1.domain.name.  your\.mail.domain.name.  ( 1 1h 5m 1d 5m )
```

### 8.6 PTR Record

Performs a reverse DNS lookup, resolving an IP to a DNS name. The target must point to an A or AAAA record.

```Bash
# Usage: <REMAINING_ADDRESS>                PTR  <TARGET>.

# IPv4 Example - for zone 30.20.10.in-addr.arpa
10                                  PTR  www.domain.name.

# IPv6 Example - for zone 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa
0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.1     PTR  www.domain.name.
```

### 8.7 MX Record

Specifies the mail server responsible for accepting emails. You must configure a corresponding A or AAAA record for the target MX!

```Bash
# Syntax: <NAME>          MX <PRIORITY>   <TARGET>
domain.name     MX 10           mail.domain.name.
```

### 8.8 SRV Record

Defines the location (hostname and port) of specific services, primarily used for protocols like Active Directory, SIP, or Minecraft.

```Bash
# Syntax: _<service>._<proto>.<NAME>          SRV <PRIORITY> <WEIGHT> <PORT> <TARGET>
_minecraft._tcp.mc1.domain.name     SRV 10          0       25565   srv4.domain.name.
```

## 9. Zone File Management

### 9.1 AppArmor Configuration

Bind9 is strictly contained by AppArmor on Debian 13 Trixie. If you place configuration or zone files outside of standard directories like `/etc/named/` or `/var/lib/bind/` (e.g., in `/storage/dns/`), Bind9 will generate a *permission denied* error even if directory permissions are `777`.

You must modify the AppArmor profile to grant the daemon read and write access to custom paths.

```Bash
# Edit or create /etc/apparmor.d/local/usr.sbin.named
# Add the following directives:
/storage/dns/ r,
/storage/dns/** rw,

# Reload the AppArmor profile for the changes to take effect
apparmor_parser -r /etc/apparmor.d/usr.sbin.named
```

**Command Breakdown & Explanation:**

- `/storage/dns/ r,`: Grants read-only access to the top-level directory.
- `/storage/dns/** rw,`: Grants recursive read and write permissions to all contents within the directory.
- `apparmor_parser -r`: Reloads the policy to apply the new local overrides immediately.

### 9.2 File Setup

#### 9.2.1 By hand

Manually creating zone files including the required SOA and NS records. Ensure you increment the Serial number upon any edits, otherwise secondary servers will ignore the changes.

```Bash
# Forward lookup zone - /var/lib/bind/domain.name.hosts
$TTL 1d
$ORIGIN domain.name.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.
@   MX      10  ns.domain.name.

ns  A       10.20.30.10
ns  AAAA    2001:db8:1010:1010::1010
www CNAME   ns.domain.name.
```

```Bash
# IPv4 Reverse lookup zone - /var/lib/bind/30.20.10.in-addr.arpa
$TTL 1d
$ORIGIN 30.20.10.in-addr.arpa.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.

10  PTR     ns.domain.name.
```

```Bash

# IPv6 Reverse lookup zone - /var/lib/bind/0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa
$TTL 1d
$ORIGIN 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.

0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.1     PTR  ns.domain.name.
```

#### 9.2.2 SOA from file

If you need a quick empty zone template (similar to `db.empty` from older Debian versions), you can extract it directly from the local manual.

> [!NOTE]
> This requires the `bind9-doc` package to be installed.

```Bash
# Extract the default SOA template
grep -A 11 "; default TTL for zone" /usr/share/doc/bind9-doc/arm/chapter3.html | awk -F'</span>' '{print $2}' > /etc/bind/db.empty
```

**Command Breakdown & Explanation:**

- `grep -A 11`: Finds the target string in the HTML manual and grabs the 11 trailing lines.
- `awk`: Cleans up the HTML formatting tags to return pure text.
- `> /etc/bind/db.empty`: Writes the parsed template into a reusable file block.

## 10. Verification and Troubleshooting

> [!NOTE]
> Validate the service statuses, network port bindings, and configurations on Debian 13 using standard diagnostic tools. Ensure port 53 is not conflicting with `systemd-resolved`, which is common on modern Linux systems. Modern clients (like Windows 11) will often fall back to alternative DNS (or DoH) if basic local resolution fails, making strict local verification paramount.

### 10.1 Verify Bind9 service status

**Command:** `systemctl status bind9`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/bind9.service)`

### 10.2 Verify network listening state on port 53

**Command:** `ss -tuln | grep :53`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:53`, `0.0.0.0:53`, or explicitly configured IPs like `10.20.30.10:53`

### 10.3 Verify zone configuration syntax

**Command:** `named-checkconf` and `named-checkzone company.com /var/lib/bind/company.com.in`

**What it checks and variables to look for:**

- **named-checkconf output**: Should return empty (indicating no syntax errors found in the configurations).
- **named-checkzone output**: Must specifically state `OK` for the specified zone file, confirming the SOA and formatting are valid.

<!-- Created by: Gergő Téringer, 2026 -->
