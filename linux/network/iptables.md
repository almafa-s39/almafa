<!-- 
---
title: "iptables"
author: "Gergő Téringer"
---
 -->
# IPTables

This document provides administrative procedures for configuring a stateful firewall and routing using the legacy `iptables` framework on Debian 13 (Trixie). It replicates the previously defined `nftables` behavior, translating the rules into standard `iptables` and `ip6tables` commands, and utilizes the `iptables-persistent` package to ensure rules survive system reboots.

> [!NOTE]
> Unlike `nftables`, `iptables` does not support native declarative configuration files or named sets out-of-the-box. Firewalls are typically deployed by executing a Bash script containing the rules sequentially, which are then saved to persistence files.

## 1. Package Installation and Service Management

To enable iptables rule persistence across reboots, you must install the `iptables-persistent` package.

> [!IMPORTANT]
> During the installation of `iptables-persistent`, an interactive wizard will prompt you to save current IPv4 and IPv6 rules. You can select **Yes** or **No**, as we will manually overwrite the persistence files later in this guide.

```Bash
# Install the core iptables and persistence packages
apt install iptables iptables-persistent

# Enable the persistent netfilter service to restore rules on boot
systemctl enable netfilter-persistent.service --now
```

**Command Breakdown & Explanation:**

- `apt install iptables iptables-persistent`: Installs the legacy Netfilter user-space utility and the `netfilter-persistent` daemon which reads `/etc/iptables/rules.v4` and `rules.v6` on startup.

## 2. IP Packet Forwarding Configuration

Routing packets between network interfaces requires kernel IP forwarding to be enabled.

```Bash
# Enable IPv4 and IPv6 packet forwarding in sysctl
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/' /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `sysctl -p`: Instructs the kernel to dynamically reload configurations from the `sysctl.conf` file without requiring a reboot.

## 3. Deploying Iptables via Bash Script

Because `iptables` lacks native named sets, variables must be expanded manually or iterated via Bash loops. The following script flushes existing rules, sets the default drop policy, and translates your previous `nftables` logic directly into kernel memory.

```Bash
# Create the firewall deployment script
cat << 'EOF' > /root/deploy-firewall.sh
#!/bin/bash

# 1. Flush all existing rules and custom chains
iptables -F
iptables -t nat -F
iptables -X
ip6tables -F
ip6tables -t nat -F
ip6tables -X

# 2. Apply default DROP policy for the FORWARD chain
iptables -P FORWARD DROP
ip6tables -P FORWARD DROP

# 3. Accept established/related return traffic
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 4. Allow traffic from INT (10.1.10.0/24) and DMZ (10.1.20.0/24) to internet
iptables -A FORWARD -s 10.1.10.0/24 -o ens18 -j ACCEPT
iptables -A FORWARD -s 10.1.20.0/24 -o ens18 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:10::/64 -o ens18 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:20::/64 -o ens18 -j ACCEPT

# 5. Allow traffic from INT to DMZ
iptables -A FORWARD -s 10.1.10.0/24 -d 10.1.20.0/24 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:10::/64 -d 2001:db8:1001:20::/64 -j ACCEPT

# 6. Allow traffic from VPN clients (10.1.30.0/24) to DMZ and INT
iptables -A FORWARD -s 10.1.30.0/24 -d 10.1.10.0/24 -j ACCEPT
iptables -A FORWARD -s 10.1.30.0/24 -d 10.1.20.0/24 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:30::/64 -d 2001:db8:1001:10::/64 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:30::/64 -d 2001:db8:1001:20::/64 -j ACCEPT

# 7. Allow mail server to reach LDAP/LDAPS
iptables -A FORWARD -s 10.1.20.10 -d 10.1.10.10 -p tcp -m multiport --dports 389,636 -j ACCEPT
ip6tables -A FORWARD -s 2001:db8:1001:20::10 -d 2001:db8:1001:10::10 -p tcp -m multiport --dports 389,636 -j ACCEPT

# 8. Configure PAT (Masquerade) for outbound INT and DMZ networks
iptables -t nat -A POSTROUTING -s 10.1.10.0/24 -o ens18 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.1.20.0/24 -o ens18 -j MASQUERADE

# 9. Create port forwarding (DNAT) rules for external HTTP(S) and DNS
iptables -t nat -A PREROUTING -d 1.1.1.10 -p tcp -m multiport --dports 53,80,443 -j DNAT --to-destination 10.1.20.20
iptables -t nat -A PREROUTING -d 1.1.1.10 -p udp --dport 53 -j DNAT --to-destination 10.1.20.20

# 10. Route INT and VPN networks to transparent HTTP proxy (Port 3128)
iptables -t nat -A PREROUTING -s 10.1.10.0/24 -p tcp --dport 80 -j REDIRECT --to-ports 3128
iptables -t nat -A PREROUTING -s 10.1.30.0/24 -p tcp --dport 80 -j REDIRECT --to-ports 3128
ip6tables -t nat -A PREROUTING -s 2001:db8:1001:10::/64 -p tcp --dport 80 -j REDIRECT --to-ports 3128
ip6tables -t nat -A PREROUTING -s 2001:db8:1001:30::/64 -p tcp --dport 80 -j REDIRECT --to-ports 3128
EOF

# Make the script executable and run it to inject rules into the active kernel
chmod +x /root/deploy-firewall.sh
/root/deploy-firewall.sh
```

**Command Breakdown & Explanation:**

- `-F` and `-X`: Empties all chains and deletes custom chains to prevent rule duplication on execution.
- `-P FORWARD DROP`: Establishes the default security posture for traversing traffic.
- `-m conntrack --ctstate ESTABLISHED,RELATED`: The `iptables` equivalent of stateful connection tracking.
- `-m multiport --dports 389,636`: Loads the `multiport` extension module to allow comma-separated port matching on a single line.
- `-j MASQUERADE`: Triggers dynamic Source NAT.
- `-j DNAT --to-destination`: Modifies the destination IP in the packet header before it reaches the routing decision logic.
- `-j REDIRECT --to-ports`: Transparently modifies destination port routing to point to a service on the local firewall host.

## 4. Saving Rules for Persistence

Rules executed in Bash are completely lost on reboot unless exported to the `iptables-persistent` directory structure.

```Bash
# Export the active IPv4 memory state to the persistent rules file
iptables-save > /etc/iptables/rules.v4

# Export the active IPv6 memory state to the persistent rules file
ip6tables-save > /etc/iptables/rules.v6
```

**Command Breakdown & Explanation:**

- `iptables-save > /etc/iptables/rules.v4`: Dumps all actively running IPv4 kernel rules into the standardized file structure read by the `netfilter-persistent` daemon during the Debian boot sequence.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the persistence service status, active `filter` rules, and active `nat` rules on Debian 13 using standard administrative tools.

### 5.1 Verify persistence daemon service status

**Command:** `systemctl status netfilter-persistent`

**What it checks and variables to look for:**

- **Active**: Must be `active (exited)` (standard behavior for oneshot boot-loading services)
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/netfilter-persistent.service; enabled)`

### 5.2 Verify active IPv4 filter rules (FORWARD Chain)

**Command:** `iptables -L FORWARD -v -n`

**What it checks and variables to look for:**

- **Policy**: Must state `Chain FORWARD (policy DROP)`
- **Ruleset**: Must display the individual ACCEPT statements applied by your deployment script, including source and destination IP matching blocks

### 5.3 Verify active IPv4 NAT rules

**Command:** `iptables -t nat -L -v -n`

**What it checks and variables to look for:**

- **Chain PREROUTING**: Must list the `DNAT` and `REDIRECT` targets
- **Chain POSTROUTING**: Must list the `MASQUERADE` targets tied to the `ens18` outbound interface

<!-- Created by: Gergő Téringer, 2026 -->