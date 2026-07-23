<!-- 
---
title: "VLAN Subinterfaces on Debian"
author: "Gergő Téringer"
---
 -->
# VLAN Subinterfaces on Debian

This document covers creating 802.1Q VLAN subinterfaces on Debian, both
temporarily with `ip link` and persistently via `/etc/network/interfaces`
(ifupdown). The connected switch port must already be configured as a
trunk carrying the relevant VLAN tag(s).

## 1. Install Prerequisites

```bash
apt install vlan
modprobe 8021q
echo "8021q" >> /etc/modules
```

**Command Breakdown & Explanation:**

- `apt install vlan`: Installs `vconfig` and related tooling. The `ip`
  command itself works without this package, but installing it is still
  the conventional way to pull in `8021q` support and documentation.
- `modprobe 8021q`: Loads the kernel VLAN tagging module for the current
  session.
- `echo "8021q" >> /etc/modules`: Ensures the module is loaded
  automatically on every boot.

> [!NOTE]
> The parent (physical) interface should generally be left with no IP
> address (`inet manual`) when it is only being used as a VLAN trunk.

## 2. Temporary Subinterface (ip command)

```bash
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 192.168.10.1/24 dev eth0.10
ip link set eth0.10 up
```

**Command Breakdown & Explanation:**

- `ip link add link eth0 name eth0.10 type vlan id 10`: Creates VLAN
  subinterface `eth0.10`, tagged with VLAN ID `10`, on parent interface
  `eth0`.
- `ip addr add 192.168.10.1/24 dev eth0.10`: Assigns an IPv4 address to
  the subinterface.
- `ip link set eth0.10 up`: Brings the subinterface online.

> [!WARNING]
> This method does not survive a reboot. Use Section 3 for a persistent
> configuration.

## 3. Persistent Subinterface (/etc/network/interfaces)

```bash
auto eth0.10
iface eth0.10 inet static
    address 192.168.10.1
    netmask 255.255.255.0
    vlan-raw-device eth0
```

**Command Breakdown & Explanation:**

```markdown
- `auto eth0.10`: Brings the interface up automatically at boot.
- `iface eth0.10 inet static`: Declares a statically addressed
  interface named `eth0.10`. The `.10` suffix must match the VLAN ID.
- `vlan-raw-device eth0`: Tells ifupdown which physical interface
  carries the tagged traffic for this VLAN subinterface.
```

```bash
ifup eth0.10
```

> [!TIP]
> The VLAN ID does not have to be encoded in the interface name (e.g.
> `vlan10` also works), but if the name doesn't end in `.<id>`, the
> `vlan-raw-device` line becomes mandatory rather than optional.

## 4. Verification and Troubleshooting

### 4.1 Verify the subinterface and VLAN ID

```bash
ip -d link show eth0.10
```

What it checks and variables to look for:

- **state**: Must be `UP`.
- **vlan protocol / id**: Must show `802.1q` and the expected VLAN ID
  (e.g., `id 10`), confirming the tag matches what the switch port
  expects.

### 4.2 Verify the 8021q module is loaded

```bash
lsmod | grep 8021q
```

What it checks and variables to look for:

- **Output**: A line beginning with `8021q` must be present. No output
  means the module isn't loaded — re-run `modprobe 8021q`.

### 4.3 Verify assigned address and connectivity

```bash
ip addr show eth0.10
ping -c 3 -I eth0.10 192.168.10.254
```

What it checks and variables to look for:

- **inet**: Must show the expected IPv4 address/prefix on `eth0.10`.
- **ping output**: `0% packet loss` confirms tagged frames are reaching
  and returning from a host on the same VLAN; total failure usually
  means the switch port isn't trunking this VLAN ID.

<!-- Created by: Gergő Téringer, 2026 -->