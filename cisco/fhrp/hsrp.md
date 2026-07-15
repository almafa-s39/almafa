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
