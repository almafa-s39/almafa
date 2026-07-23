<!-- 
---
title: "Virtual Router Redundancy Protocol (VRRP)"
author: "Gergő Téringer"
---
 -->
# Virtual Router Redundancy Protocol (VRRP)

This document details the deployment and verification of the Virtual Router Redundancy Protocol (VRRP), an open standard first-hop redundancy protocol. It covers interface-level configurations for both IPv4 and IPv6, secure authentication, and integration with object tracking to ensure high availability for endpoint devices.

> [!IMPORTANT]
> Modern Context & Compatibility: In modern enterprise networks, ensuring high availability at the default gateway level is critical. Modern operating systems, including Windows 11 and Windows Server 2025, rely on continuous network access to maintain connections to cloud services, Active Directory, and SD-WAN overlays. Using VRRP (an open standard alternative to Cisco's proprietary HSRP) ensures non-disruptive failover across multi-vendor environments, keeping these essential connections alive without forcing ARP cache timeouts on the endpoint clients.

## 1. Interface Level Configuration

```markdown
The following configuration establishes VRRP on a VLAN interface, defining the virtual IP, priority for the active router election, preemption, authentication, and object tracking.
```

```cisco
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 vrrp 10 ip 10.10.10.1
 vrrp 10 priority 110
 vrrp 10 preempt delay minimum 10
 vrrp 10 authentication md5 key-string Passw0rd!
 vrrp 10 description LAN-GW
 vrrp 10 track 10 decrement 20
```

**Command Breakdown & Explanation:**

- `vrrp 10 ip 10.10.10.1`: Sets the Virtual IP (VIP) address that clients will use as their default gateway. `10` is the VRRP group number.
- `vrrp 10 priority 110`: Determines the Master (Active) router. The router with the highest priority becomes the Master. The default priority is `100`, so `110` ensures this router is preferred.
- `vrrp 10 preempt delay minimum 10`: Allows this router to reclaim the Master role if it goes offline and comes back online. The `10` second delay gives routing protocols time to converge before it takes over forwarding duties.
- `vrrp 10 authentication md5 key-string Passw0rd!`: Secures VRRP hello packets with an MD5 hash to prevent rogue devices from hijacking the default gateway.
- `vrrp 10 description LAN-GW`: Assigns a descriptive name to the VRRP group for easier identification in monitoring systems.
- `vrrp 10 track 10 decrement 20`: Ties VRRP to a tracking object (like a monitored WAN link). If track `10` goes down, the router automatically subtracts `20` from its priority (`110 - 20 = 90`). Because `90` is lower than the default VRRP priority of `100`, the Backup router safely and instantly takes over.

**Practical Example:**
If a company has a primary core switch and a secondary core switch, configuring VRRP ensures that end-user devices always have a reachable default gateway. If the primary switch loses power, the secondary switch detects the missing VRRP hello packets and assumes control of the `10.10.10.1` IP address and its associated virtual MAC address within seconds, resulting in minimal noticeable downtime for the users.

## 2. IPv6 Interface Level Configuration

VRRP fully supports IPv6. Similar to IPv4 operations, it is best practice to allow VRRP to automatically generate the virtual IPv6 link-local address, which clients will learn via Router Advertisements (RA).

```cisco
interface Vlan10
 ipv6 address 2001:DB8:10:10::2/64
 vrrp 10 ipv6 autoconfig
 vrrp 10 priority 110
 vrrp 10 preempt delay minimum 10
```

**Command Breakdown & Explanation:**

- `ipv6 address 2001:DB8:10:10::2/64`: Assigns the physical global unicast IPv6 address to the interface.
- `vrrp 10 ipv6 autoconfig`: Instructs VRRP to automatically generate a virtual IPv6 link-local address based on the VRRP group number.
- `vrrp 10 priority 110` and `vrrp 10 preempt ...`: Functions exactly the same as in IPv4, ensuring this router takes the Master role for IPv6 traffic and safely reclaims it after a hardware or link failure.

## 3. Troubleshooting and Verification

Verifying VRRP involves checking the master and backup states, confirming the virtual IP and MAC addresses, and ensuring that preemption and object tracking are functioning as expected.

### 3.1 Verifying Quick Status (IPv4 and IPv6)

This command provides a high-level overview of all VRRP groups on the router, displaying the local state, master router, backup router, and virtual IP.

**Command:** `show vrrp brief`

**Command Breakdown & Explanation:**
Provides a quick tabular view of VRRP instances, making it the fastest way to verify if the router is currently the Master or Backup gateway.

What it checks and variables to look for:

- **Interface**: Must be the local interface where VRRP is applied (e.g., `Vl10`).
- **Grp**: Must match the configured VRRP group number (e.g., `10`).
- **Pri**: Must show the current priority of the router. If tracking is active and the tracked object is down, you will see the decremented priority here.
- **State**: Must be `Master` on the primary gateway and `Backup` on the secondary. `Init` or `Learn` are transient states during the election process.
- **Master**: Must show the physical IP address of the current Master router (shows `local` if this router is the Master).
- **Virtual IP**: Must be the shared default gateway IP address (this will display the auto-configured link-local addresses for IPv6 groups).

### 3.2 Verifying Detailed Parameters & Object Tracking

Use this command to dig deeper into timers, authentication, virtual MAC addresses, and object tracking status.

**Command:** `show vrrp`

**Command Breakdown & Explanation:**
Displays comprehensive VRRP details, critical for diagnosing flapping elections, timer mismatches, or failed track objects.

What it checks and variables to look for:

- **State**: Confirms the exact state (e.g., `Master`) and how many state changes have occurred, which is useful for identifying flapping links or unstable elections.
- **Virtual MAC address is**: Verifies the MAC address being used. For VRRP, it typically ends in the hex equivalent of the group number (e.g., `0000.5E00.010A` for group 10).
- **Preemption enabled**: Must indicate `TRUE` or `enabled` if the preempt feature is active and displays the configured delay timer.
- **Master Router**: Must show the IP and expiration timer for the active router's hello packets.
- **Tracking**: Must display the track object ID, its current state (`Up` or `Down`), and the decrement value. This confirms if your IP SLA tracking is properly linked to the VRRP process and actively modifying the priority.

<!-- Created by: Gergő Téringer, 2026 -->