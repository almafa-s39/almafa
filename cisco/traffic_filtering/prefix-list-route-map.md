<!-- 
---
title: "Route Filtering with Prefix-Lists and Route-Maps"
author: "Gergő Téringer"
---
 -->
# Route Filtering with Prefix-Lists and Route-Maps

This document outlines the standard operating procedures for implementing granular route filtering and path manipulation using IP Prefix-Lists and Route-Maps across OSPF, EIGRP, and BGP routing protocols.

> [!IMPORTANT]
> Modern Context & Compatibility: In modern enterprise networks—especially those integrating with hybrid cloud environments (like Azure ExpressRoute or AWS Direct Connect)—strict route filtering is critical. Accidentally advertising a default route or a broad summary block into an external peer can hijack traffic, disrupting critical services like Windows Server 2025 Active Directory replication, SD-WAN control planes, or Microsoft 365 connectivity. Using Prefix-Lists and Route-Maps over legacy Access Control Lists (ACLs) provides the required exact-match precision and high-performance processing needed by modern Cisco IOS-XE control planes.

## 1. Defining IP Prefix-Lists

Prefix-lists are used to match routing updates based on the exact network prefix and subnet mask length. They are significantly more efficient and easier to read than wildcard-mask-based ACLs.

```cisco
ip prefix-list BRANCH-SUBNETS seq 10 permit 192.168.10.0/24
ip prefix-list BRANCH-SUBNETS seq 20 permit 192.168.20.0/24 le 28
ip prefix-list DEFAULT-ROUTE seq 10 permit 0.0.0.0/0
```

**Command Breakdown & Explanation:**

- `ip prefix-list BRANCH-SUBNETS seq 10 permit 192.168.10.0/24`: Matches the exact `192.168.10.0` network with a strict `/24` subnet mask.
- `le 28`: The "less than or equal to" operator. Sequence 20 matches any subnet within the `192.168.20.0/24` range that has a subnet mask length between `/24` and `/28` inclusive (e.g., `/25` or `/27` would match).
- `ip prefix-list DEFAULT-ROUTE seq 10 permit 0.0.0.0/0`: Explicitly matches the exact default route.

## 2. Configuring Route-Maps

Route-maps operate like "If/Then" programming statements. They use prefix-lists as the "If" condition (the `match` statement) and apply a "Then" action (the `set` statement). Every route-map contains an implicit deny at the end.

```cisco
route-map FILTER-BRANCH deny 10
 match ip address prefix-list DEFAULT-ROUTE
 exit
route-map FILTER-BRANCH permit 20
 match ip address prefix-list BRANCH-SUBNETS
 set metric 100 10 255 1 1500
 exit
route-map FILTER-BRANCH permit 30
 exit
```

**Command Breakdown & Explanation:**

- `route-map FILTER-BRANCH deny 10`: Sequence 10 actively drops any route that matches the condition.
- `match ip address prefix-list DEFAULT-ROUTE`: The condition for sequence 10; blocks the default route from being advertised or received.
- `route-map FILTER-BRANCH permit 20`: Allows routes matching the subsequent condition and applies an optional modification.
- `set metric ...`: Manipulates the EIGRP metric (Bandwidth, Delay, Reliability, Load, MTU) for the matched `BRANCH-SUBNETS` routes.
- `route-map FILTER-BRANCH permit 30`: An empty permit statement. This acts as a "catch-all" to permit all other routes that were not explicitly matched by earlier sequences, overriding the implicit deny.

## 3. Applying to Routing Protocols

Route-maps and Prefix-lists must be applied within the specific routing protocol configuration mode to take effect. The direction (`in` or `out`) determines whether you are filtering routes entering your routing table or routes being advertised to a neighbor.

### 3.1 OSPF Inbound Filtering

```cisco
router ospf 1
 distribute-list route-map FILTER-BRANCH in
```

**Command Breakdown & Explanation:**

- `distribute-list ... in`: In OSPF, you cannot strictly filter outbound LSAs within a single area. However, applying a distribute-list inbound prevents the matched routes from being installed into the local router's global routing table (even though they remain in the OSPF LSDB).

### 3.2 EIGRP Outbound Filtering

```cisco
router eigrp 100
 network 192.168.10.0 0.0.0.255
 distribute-list route-map FILTER-BRANCH out GigabitEthernet0/1
```

**Command Breakdown & Explanation:**

- `distribute-list ... out`: EIGRP is a distance-vector protocol, allowing for strict outbound filtering. This prevents the matched routes from ever being advertised out of the `GigabitEthernet0/1` interface, saving bandwidth and enforcing routing boundaries.

### 3.3 BGP Route-Map Integration

```cisco
router bgp 65000
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 route-map FILTER-BRANCH out
```

**Command Breakdown & Explanation:**

- `neighbor ... route-map ... out`: Applies the route-map directly to a specific BGP peer. In this example, BGP evaluates all outbound route advertisements sent to `10.0.0.2` against the `FILTER-BRANCH` route-map.

## 4. Troubleshooting and Verification

### 4.1 Verify Prefix-List Hits

Displays the configured prefix-lists and shows counters indicating how many times a specific sequence line was matched by routing updates.

**Command:** `show ip prefix-list detail`

**Command Breakdown & Explanation:**
Validates the precise syntax of the prefix-list and confirms whether dynamic routing updates are actively triggering the permit or deny conditions.

What it checks and variables to look for:

- **Prefix-list**: Must match your configured name (e.g., `BRANCH-SUBNETS`).
- **hit count**: Should be `> 0` if routes are successfully being matched by this specific sequence.
- **mask length**: Verifies `le` (less than or equal to) and `ge` (greater than or equal to) parameters are correct.

### 4.2 Verify Route-Map Configuration

Displays the configured route-maps, their sequence logic, and the associated match/set clauses.

**Command:** `show route-map FILTER-BRANCH`

**Command Breakdown & Explanation:**
Ensures that the route-map logic flows correctly, verifying that `deny` or `permit` statements are bound to the correct IP prefix-lists and that the intended metric manipulation is present.

What it checks and variables to look for:

- **route-map**: Must be `FILTER-BRANCH`.
- **Match clauses**: Must display `ip address (prefix-lists): [your-prefix-list-name]`.
- **Set clauses**: Must display the correct manipulated variables (e.g., `metric 100 10 255 1 1500` for EIGRP).

### 4.3 Verify BGP Applied Route-Maps

Displays the routes currently being advertised to a specific BGP neighbor after the outbound route-map has processed them.

**Command:** `show ip bgp neighbors 10.0.0.2 advertised-routes`

**Command Breakdown & Explanation:**
Checks the actual routing payload that is successfully leaving the router toward the peer. This is the definitive way to prove your route-map is successfully filtering unwanted prefixes (like the default route) before they reach the remote AS.

What it checks and variables to look for:

- **Network**: The default route `0.0.0.0` must NOT be present in this list.
- **Next Hop**: Must show the appropriate exit interface IP or self-originated IP.
- **Metric / LocPrf**: Must reflect any modifications applied by the route-map's `set` commands.

<!-- Created by: Gergő Téringer, 2026 -->