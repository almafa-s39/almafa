<!-- 
---
title: "HSRP"
author: "Gergő Téringer"
---
 -->
# HSRP

Hot Standby Router Protocol (HSRP) is a Cisco-proprietary First-Hop Redundancy Protocol (FHRP) that provides high availability for IP networks. It uses an active/standby model where one router actively forwards traffic for a virtual IP address, and another router remains in standby mode, ready to take over if the active router fails.

## 1. Interface Level Configuration

The following configuration establishes HSRP on a VLAN interface, defining the virtual IP, priority for the active router election, preemption, authentication, and object tracking.

**Configuration:**

```cisco
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt delay minimum 10
 standby 10 authentication md5 key-string Passw0rd!
 standby 10 name LAN-GW
 standby 10 track 10 decrement 20
```

**Command Breakdown & Explanation:**

- `standby 10 ip 10.10.10.1`: Sets the Virtual IP (VIP) address that clients will use as their default gateway. `10` is the HSRP group number.
- `standby 10 priority 110`: Determines the Active router. The router with the highest priority becomes Active. The default priority is `100`, so `110` ensures this router is preferred.
- `standby 10 preempt delay minimum 10`: Allows this router to reclaim the Active role if it goes offline and comes back online. The `10` second delay gives routing protocols time to converge before it takes over forwarding duties.
- `standby 10 authentication md5 key-string Passw0rd!`: Secures HSRP hello packets with an MD5 hash to prevent rogue devices from hijacking the default gateway.
- `standby 10 name LAN-GW`: Assigns a descriptive name to the HSRP group for easier identification in monitoring systems.
- `standby 10 track 10 decrement 20`: Ties HSRP to a tracking object (like a monitored WAN link). If track `10` goes down, the router automatically subtracts `20` from its priority (`110 - 20 = 90`). Because `90` is lower than the default standby priority of `100`, the standby router safely and instantly takes over.

**Practical Example:**
If a company has a primary core switch and a secondary core switch, configuring HSRP ensures that end-user devices always have a reachable default gateway. If the primary switch loses power, the secondary switch detects the missing HSRP hello packets and assumes control of the `10.10.10.1` IP address and its associated virtual MAC address within seconds, resulting in minimal noticeable downtime for the users.

## 2. IPv6 Interface Level Configuration

HSRP fully supports IPv6. Similar to GLBP, it is best practice to allow HSRP to automatically generate the virtual IPv6 link-local address, which clients will learn via Router Advertisements (RA). HSRP for IPv6 requires HSRP version 2.

**Configuration:**

```cisco
interface Vlan10
 ipv6 address 2001:DB8:10:10::2/64
 standby version 2
 standby 10 ipv6 autoconfig
 standby 10 priority 110
 standby 10 preempt delay minimum 10
```

**Command Breakdown & Explanation:**

- `ipv6 address 2001:DB8:10:10::2/64`: Assigns the physical global unicast IPv6 address to the interface.
- `standby version 2`: Enables HSRP version 2, which is required for IPv6 support and allows for a larger range of group numbers and millisecond timers.
- `standby 10 ipv6 autoconfig`: Instructs HSRP to automatically generate a virtual IPv6 link-local address based on the HSRP group number.
- `standby 10 priority 110` and `standby 10 preempt ...`: Functions exactly the same as in IPv4, ensuring this router takes the Active role for IPv6 traffic and safely reclaims it after a hardware or link failure.

## 3. Troubleshooting

Verifying Hot Standby Router Protocol (HSRP) involves checking the active and standby states, confirming the virtual IP and MAC addresses, and ensuring that preemption and object tracking are functioning as expected.

### 3.1 Verifying Quick Status (IPv4 and IPv6)

This command provides a high-level overview of all HSRP groups on the router, displaying the local state, active router, standby router, and virtual IP.

**Command:** `show standby brief`

What it checks and variables to look for:

- `Interface`: The local interface where HSRP is applied (e.g., `Vl10`).
- `Grp`: The HSRP group number.
- `Pri`: The current priority of the router. If tracking is active and the tracked object is down, you will see the decremented priority here.
- `State`: The role of this router. You want to see `Active` on the primary gateway and `Standby` on the backup. `Listen` or `Speak` are transient states during the election process.
- `Active`: The physical IP address of the current Active router (shows `local` if this router is Active).
- `Standby`: The physical IP address of the backup router.
- `Virtual IP`: The shared default gateway IP address (this will display the auto-configured link-local addresses for IPv6 groups).

### 3.2 Verifying Detailed Parameters & Object Tracking

Use this command to dig deeper into timers, authentication, virtual MAC addresses, and object tracking status.

**Command:** `show standby`

What it checks and variables to look for:

- `State is`: Confirms the exact state and how many state changes have occurred, which is useful for identifying flapping links or unstable elections.
- `Virtual MAC address is`: Verifies the MAC address being used. For IPv4 HSRPv1, it ends in the group number (e.g., `0000.0c07.ac0a` for group 10).
- `Preemption enabled`: Confirms if the preempt feature is active and displays the configured delay timer.
- `Active router is`: Shows the IP and expiration timer for the active router's hello packets.
- `Standby router is`: Shows the IP of the standby router.
- `Tracking`: Displays the track object ID, its current state (`Up` or `Down`), and the decrement value. This confirms if your IP SLA tracking is properly linked to the HSRP process and actively modifying the priority.

<!-- Created by: Gergő Téringer, 2026 -->