<!-- 
---
title: "PBR(iproute2)"
author: "Gergő Téringer"
---
 -->
# PBR(iproute2)

This document provides administrative procedures for implementing Policy-Based Routing (PBR) on Debian 13 (Trixie) using the `iproute2` package. PBR allows administrators to influence routing decisions based on specific criteria such as source addresses, destination ports, or ingress interfaces, rather than relying solely on the destination IP address.

> [!NOTE]
> PBR operates by evaluating packets against a sequential list of rules (`ip rule`). When a match occurs, the packet is forwarded according to a specifically defined, secondary routing table rather than the main system routing table.

## 1. Package Installation and Custom Table Creation

The `iproute2` utility suite is typically pre-installed on Debian. To utilize PBR using named tables instead of arbitrary numbers, you must append your custom routing table identifiers to `/etc/iproute2/rt_tables`.

```Bash
# Install the core iproute2 utility package
apt install iproute2

# Register a new custom routing table named 'research' with ID 100
echo "100 research" >> /etc/iproute2/rt_tables
```

**Command Breakdown & Explanation:**

- `apt install iproute2`: Installs the standard suite of networking utilities including `ip route` and `ip rule`.
- `echo "100 research" >> ...`: Maps the arbitrary table ID `100` to the human-readable string `research`. This allows you to reference `table research` in future commands instead of remembering the numeric ID.

## 2. Defining Routes and Policy Rules

Once the custom routing table exists, you must populate it with specific route paths and then create policies (`ip rule`) that dictate which traffic should be directed to look at this table.

> [!WARNING]
> Routes and rules applied via the `ip` command are ephemeral and will not survive a system reboot. You must script these commands into a dedicated system service, a network interface hook, or a startup script to ensure persistence.

```Bash
# Insert a custom route into the 'research' table
ip route add 10.10.20.0/24 dev ipsec0 via 10.255.255.2 table research

# Insert a default route into an arbitrary table 80
ip route add default via 2.2.2.2 dev ens22 table 80

# Create policy rules matching specific criteria to use the custom tables
ip rule add from 10.10.10.0/24 lookup research prio 100
ip rule add dport 80 table 80
ip rule add dport 443 table 80
ip rule add iif ens224 table 250
```

**Command Breakdown & Explanation:**

- `ip route add ... table research`: Installs a static route directing traffic destined for `10.10.20.0/24` out the `ipsec0` interface through the gateway `10.255.255.2`, but isolates this route exclusively to the `research` table.
- `ip route add ... table 80`: Demonstrates inserting a route on-the-fly to a table using its numeric ID without mapping it in `rt_tables`.
- `ip rule add from ... lookup research`: Instructs the kernel to intercept packets originating from the `10.10.10.0/24` subnet and route them using the paths defined in the `research` table.
- `prio 100`: Assigns an explicit evaluation priority to the rule (lower numbers are evaluated first).
- `dport 80` / `dport 443`: Matches traffic based on destination TCP/UDP ports (useful for isolating HTTP/HTTPS traffic to a specific gateway).
- `iif ens224`: Matches traffic based on the ingress (incoming) network interface.

## 3. Verification and Troubleshooting

> [!NOTE]
> Validate the active routing rules and isolated routing table contents on Debian 13 using standard `iproute2` diagnostic commands.

### 3.1 Verify active routing policies (rules)

**Command:** `ip rule show`

**What it checks and variables to look for:**

- **Priority**: Must list your configured priority numbers (e.g., `100:`)
- **Match criteria**: Must display the correct source IP, destination port, or interface match conditions
- **Lookup table**: Must indicate the targeted custom table (e.g., `lookup research` or `lookup 80`)

## 3.2 Verify routes within a custom table

**Command:** `ip route show table research` *(or substitute `research` with the target table number)*

**What it checks and variables to look for:**

- **Destination network**: Must display the targeted subnet (e.g., `10.10.20.0/24` or `default`)
- **Next-hop gateway**: Must output the defined `via`

<!-- Created by: Gergő Téringer, 2026 -->