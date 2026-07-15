# GLBP

Gateway Load Balancing Protocol (GLBP) is a Cisco-proprietary protocol that provides both high availability and load balancing for first-hop routers. Unlike HSRP or VRRP, which operate in an active/standby model, GLBP actively utilizes all redundant routers in the group by answering ARP requests with different virtual MAC addresses.

## 1. IPv4 Interface Level Configuration

The following configuration establishes GLBP on a VLAN interface, defining the Active Virtual Gateway (AVG) election process, security, and advanced tracking to ensure traffic only flows through healthy uplinks.

**Configuration:**

```cisco
interface Vlan103
 ip address 10.10.103.2 255.255.255.0
 glbp 103 ip 10.10.103.1
 glbp 103 priority 110
 glbp 103 preempt [ delay minimum 10 ]
 glbp 103 load-balancing host-dependent
 glbp 103 authentication md5 key-string Passw0rd!
 glbp 103 name MGMT
 glbp 103 weighting track 10 decrement 20
 glbp 103 forwarder preempt [ delay minimum 10 ]
end
```

**Command Breakdown & Explanation:**

- `glbp 103 ip 10.10.103.1`: Sets the Virtual IP (VIP) address that clients will use as their default gateway. `103` is the GLBP group number.
- `glbp 103 priority 110`: Determines the Active Virtual Gateway (AVG). The router with the highest priority becomes the AVG, which is responsible for answering client ARP requests. The default priority is `100`.
- `glbp 103 preempt delay minimum 10`: Allows this router to reclaim the AVG role if it goes offline and comes back, assuming it has a higher priority than the current active router. The `10` second delay ensures routing protocols have time to converge before it takes over.
- `glbp 103 load-balancing host-dependent`: Instructs the AVG to always return the exact same virtual MAC address to a specific client MAC address. This is crucial if you have stateful firewalls or NAT upstream, ensuring a client's traffic consistently flows through the same physical router. (Other options include `round-robin` or `weighted`).
- `glbp 103 authentication md5 key-string Passw0rd!`: Secures GLBP hello packets with an MD5 hash to prevent rogue devices from joining the redundancy group and hijacking gateway traffic.
- `glbp 103 name MGMT`: Assigns a descriptive name to the GLBP group for easier management.
- `glbp 103 weighting track 10 decrement 20`: Ties the router's forwarding capability to an object track (for example, tracking the state of an outbound WAN interface). If track `10` goes down, this router's GLBP weighting is reduced by `20`. If the weight falls below a configured lower threshold, it stops acting as an Active Virtual Forwarder (AVF) and safely shifts client traffic to the other router.
- `glbp 103 forwarder preempt delay minimum 10`: Similar to the AVG preempt, this allows the router to reclaim its AVF traffic-forwarding role once its tracked interface recovers and its weighting is restored.

**Practical Example:**
If you have two distribution switches handling VLAN 103, HSRP would leave one switch's uplink sitting completely idle. With GLBP configured, the AVG switch intercepts all ARP requests for `10.10.103.1`. It gives half of your user PCs the virtual MAC of Switch A, and the other half the virtual MAC of Switch B. Both uplinks are utilized simultaneously, effectively doubling your bandwidth while maintaining sub-second redundancy if one switch fails.

## IPv6

Gateway Load Balancing Protocol (GLBP) fully supports IPv6, providing the same active-active load balancing and redundancy for IPv6 networks as it does for IPv4. The configuration syntax is highly similar, primarily replacing the `ip` keyword with `ipv6` and utilizing IPv6 link-local addresses for the virtual gateway.

### 1. IPv6 Interface Level Configuration

In this configuration, GLBP is enabled for IPv6 on the interface. Instead of manually specifying a Virtual IP, it is a best practice in IPv6 to let GLBP automatically generate a link-local virtual IPv6 address using the `autoconfig` parameter.

*Configuration:**

```cisco
interface Vlan103
 ipv6 address 2001:DB8:10:103::2/64
 glbp 103 ipv6 autoconfig
 glbp 103 priority 110
 glbp 103 preempt delay minimum 10
 glbp 103 load-balancing host-dependent
 glbp 103 name MGMT-v6
 glbp 103 weighting track 10 decrement 20
```

**Command Breakdown & Explanation:**

- `ipv6 address 2001:DB8:10:103::2/64`: Assigns the physical global unicast IPv6 address to the interface.
- `glbp 103 ipv6 autoconfig`: Instructs GLBP to automatically generate a virtual IPv6 link-local address for group `103`. IPv6 clients will learn this virtual gateway address dynamically via IPv6 Router Advertisement (RA) messages.
- `glbp 103 priority 110`: Elects this router as the Active Virtual Gateway (AVG) for the IPv6 group, as `110` is higher than the default `100`.
- `glbp 103 preempt delay minimum 10`: Allows the router to reclaim its AVG status after a failure, waiting `10` seconds to ensure routing protocol convergence.
- `glbp 103 load-balancing host-dependent`: Ensures that a specific IPv6 client always receives the same virtual MAC address, maintaining consistent traffic flows through a single Active Virtual Forwarder (AVF).
- `glbp 103 name MGMT-v6`: Assigns a descriptive name to the IPv6 GLBP group for easier administrative tracking.
- `glbp 103 weighting track 10 decrement 20`: Reduces the router's GLBP weight if the tracked object (like an outbound IPv6 WAN link) goes down, allowing the standby router to smoothly take over forwarding duties for IPv6 traffic.

**Practical Example:**
When an IPv6 client boots up on VLAN 103, it sends a Router Solicitation (RS) message. The GLBP AVG responds with a Router Advertisement (RA) containing the auto-configured GLBP virtual link-local address as the default gateway. When the client sends Neighbor Discovery (ND) requests for that gateway, the AVG load-balances by replying with different virtual MAC addresses, distributing the IPv6 traffic perfectly across both redundant distribution switches.
