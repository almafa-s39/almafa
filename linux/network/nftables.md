<!-- 
---
title: "nftables"
author: "Gergő Téringer"
---
 -->
# NFTables

This document provides administrative procedures for installing and configuring `nftables` on Debian 13 (Trixie). It covers enabling packet forwarding, defining stateful firewall rules, configuring Network Address Translation (NAT), using named sets, and deploying a comprehensive ruleset.

> [!NOTE]
> Nftables is the modern subsystem of the Linux kernel providing filtering and classification of network packets, replacing legacy `iptables`. The primary configuration file resides at `/etc/nftables.conf`.

## 1. Package Installation and Service Management

Debian typically includes `nftables` by default. If the system was installed via minimal ISO, the package must be installed and the service explicitly enabled.

```Bash
# Install the nftables package
apt install nftables

# Enable the service to start at boot and launch it immediately
systemctl enable nftables.service --now
```

**Command Breakdown & Explanation:**

- `apt install nftables`: Installs the core Netfilter rule management utilities.
- `systemctl enable nftables.service --now`: Configures the firewall daemon to parse `/etc/nftables.conf` automatically on boot and starts the active session.

## 2. IP Packet Forwarding Configuration

To operate the Debian system as a router or stateful firewall across multiple interfaces, IP packet forwarding must be explicitly enabled in the kernel parameters.

```Bash
# Enable IPv4 and IPv6 packet forwarding in sysctl
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/' /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `sed -i 's/.../.../'`: Locates and uncomments the default IP forwarding parameters inside `/etc/sysctl.conf`.
- `sysctl -p`: Instructs the kernel to dynamically reload configurations from the `sysctl.conf` file without requiring a reboot.

## 3. Core Concepts and Syntax Options

The `nftables` syntax relies on tables containing chains, which evaluate packets through hooks (e.g., `prerouting`, `forward`, `postrouting`).

> [!CAUTION]
> Never filter traffic (accept/drop) inside a NAT chain! NAT chains only evaluate the very first packet of a connection flow. Filtering must be strictly handled inside `filter` type chains.

### 3.1 Common Matching Expressions

Traffic matching relies on inspecting network headers and connection tracking states.

Here are the most common match parameters:

- `ip saddr <IPv4 address/prefix>`: Matches source IPv4 networks (e.g., `10.1.0.0/24`)
- `ip daddr <IPv4 address/prefix>`: Matches destination IPv4 networks
- `ip6 saddr` / `ip6 daddr`: Matches IPv6 source or destination addresses
- `tcp sport` / `tcp dport`: Matches source or destination TCP ports
- `udp sport` / `udp dport`: Matches source or destination UDP ports
- `iif` / `oif`: Matches the ingress (in) or egress (out) physical interface
- `ct state { established, related }`: Connection Tracking matching to permit return traffic for active sessions

### 3.2 Chain Actions

Rules terminate with an action. Non-final actions process data and pass the packet to the next rule, while final actions dictate the packet's immediate fate.

**Final Filter Actions:**

- `accept`: Permits the packet
- `drop`: Silently discards the packet

**Final NAT Actions:**

- `snat to <IP>[:PORT]`: Source NAT, changing the origin IP address
- `masquerade`: Dynamic SNAT, automatically adapting to the outbound interface IP
- `dnat to <IP>[:PORT]`: Destination NAT (Port Forwarding)
- `redirect to <PORT>`: Redirects incoming traffic locally to a service running on the firewall host

**Non-Final Actions:**

- `counter`: Increments a packet counter
- `log`: Writes a record of the packet to the system log

### 4. Deploying a Comprehensive Stateful Firewall Configuration

This section provides a complete, production-ready `/etc/nftables.conf` file utilizing variables (named sets), stateful connection tracking, interface routing, Masquerade NAT, and Destination NAT.

> [!TIP]
> The `flush ruleset` command at the top of the file is critical. It wipes all existing rules in kernel memory before compiling the new parameters, preventing duplicated rules on service restart.

```Bash
# Overwrite the default configuration with the comprehensive ruleset
cat << 'EOF' > /etc/nftables.conf
#!/usr/sbin/nft -f

flush ruleset

# Define named sets for network zones
define INT4 = { 10.1.10.0/24 }
define INT_DMZ4 = { $INT4, 10.1.20.0/24 }
define INT_DMZ6 = { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64 }

table inet filter {
    chain forward {
        type filter hook forward priority filter;
        # Apply default drop policy
        policy drop;
    
        # Accept return traffic using connection tracking
        ct state { established, related } accept;
    
        # Allow traffic from INT and DMZ to internet via outbound interface ens18
        ip saddr $INT_DMZ4 oif ens18 accept;
        ip6 saddr $INT_DMZ6 oif ens18 accept;
    
        # Allow traffic from INT to DMZ
        ip saddr 10.1.10.0/24 ip daddr 10.1.20.0/24 accept;
        ip6 saddr 2001:db8:1001:10::/64 ip6 daddr 2001:db8:1001:20::/64 accept;
    
        # Allow traffic from VPN clients to DMZ and INT
        ip saddr 10.1.30.0/24 ip daddr { 10.1.10.0/24, 10.1.20.0/24 } accept;
        ip6 saddr 2001:db8:1001:30::/64 ip6 daddr { 2001:db8:1001:10::/64, 2001:db8:1001:20::/64 } accept;
    
        # Allow mail server to reach LDAP/LDAPS
        ip saddr 10.1.20.10 ip daddr 10.1.10.10 tcp dport { 389, 636 } accept;
        ip6 saddr 2001:db8:1001:20::10 ip6 daddr 2001:db8:1001:10::10 tcp dport { 389, 636 } accept;
    }
  
    chain srcnat {
        type nat hook postrouting priority srcnat;
    
        # Configure PAT (Masquerade) for outbound INT and DMZ networks
        ip saddr { 10.1.10.0/24, 10.1.20.0/24 } oif ens18 masquerade;
    }
  
    chain dstnat {
        type nat hook prerouting priority dstnat;
    
        # Create port forwarding rules for external HTTP(S) and DNS traffic to DMZ host
        ip daddr 1.1.1.10 tcp dport { 53, 80, 443 } dnat to 10.1.20.20;
        ip daddr 1.1.1.10 udp dport 53 dnat to 10.1.20.20;
    
        # Transparently route INT and VPN networks to a local HTTP proxy running on port 3128
        ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 80 redirect to 3128;
        ip6 saddr { 2001:db8:1001:10::/64, 2001:db8:1001:30::/64 } tcp dport 80 redirect to 3128;
    }
}
EOF

# Load and validate the new configuration
nft -f /etc/nftables.conf
```

**Command Breakdown & Explanation:**

- `flush ruleset`: Empties all active Netfilter hooks to ensure a clean application.
- `define <name> = { ... }`: Creates variables to simplify administration of multiple IP blocks.
- `table inet filter`: Creates an address-family agnostic table (handles both IPv4 and IPv6).
- `chain forward`: Evaluates traffic passing through the router (not destined for the router host itself).
- `chain srcnat`: Post-routing chain responsible for altering outgoing traffic IPs (Masquerade).
- `chain dstnat`: Pre-routing chain responsible for Port Forwarding and proxy redirection prior to routing decisions.
- `nft -f /etc/nftables.conf`: Manually forces the `nft` engine to read and compile the target file without requiring a systemctl service restart.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the kernel IP forwarding state, nftables service status, and active parsed rulesets on Debian 13 using standard administrative tools.

### 5.1 Verify IP Forwarding kernel parameters

**Command:** `sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding`

**What it checks and variables to look for:**

- **IPv4**: Must output `net.ipv4.ip_forward = 1`
- **IPv6**: Must output `net.ipv6.conf.all.forwarding = 1`

### 5.2 Verify nftables service status

**Command:** `systemctl status nftables`

**What it checks and variables to look for:**

- **Active**: Must be `active (exited)` (this is normal for oneshot rule-loading services)
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/nftables.service; enabled)`

### 5.3 Verify active kernel ruleset and parsing

**Command:** `nft list ruleset`

**What it checks and variables to look for:**

- **Syntax Validation**: Must dump the active configuration block identical to your file structure without errors
- **Counters**: If the `counter` action was included in rules, packet and byte matches will be actively increasing

<!-- Created by: Gergő Téringer, 2026 -->