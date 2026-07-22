<!-- 
---
title: "Border Gateway Protocol (BGP) Configuration and Operations"
author: "Gergő Téringer"
---
 -->
# Border Gateway Protocol (BGP) Configuration and Operations

This document provides a comprehensive configuration, operations, and troubleshooting reference for the Border Gateway Protocol (BGP). It covers fundamental session establishment (iBGP and eBGP), advanced features such as route aggregation and communities, and the BGP Best Path Selection algorithm.

> [!IMPORTANT]
> Modern Context & Compatibility: In modern enterprise networks, BGP is the de-facto standard for hybrid-cloud connectivity (e.g., Azure ExpressRoute, AWS Direct Connect) and SD-WAN underlay/overlay fabrics. Properly configuring BGP path attributes (like Local Preference and AS_Path prepending) is critical for ensuring symmetric routing between on-premises datacenters and cloud providers, preventing asymmetric drops by stateful firewalls.

## 1. BGP Fundamentals and Session Types

BGP is a path-vector routing protocol that uses Autonomous System Numbers (ASNs) to make routing decisions. Unlike IGPs (like OSPF or EIGRP) that form adjacencies automatically via multicast, BGP requires explicit manual neighbor configurations and operates over TCP port 179.

BGP has two primary session types:

* **eBGP (External BGP):** Established between routers in *different* ASNs. By default, eBGP packets have a TTL of 1, meaning peers must be directly connected. Loop prevention is handled by the AS_Path attribute (a router drops an update if it sees its own ASN in the path).
* **iBGP (Internal BGP):** Established between routers in the *same* ASN. iBGP packets have a default TTL of 255. Loop prevention is handled by the iBGP split-horizon rule: a route learned from an iBGP peer cannot be advertised to another iBGP peer. This typically requires a full mesh of iBGP sessions or a Route Reflector.

### 1.1 Basic BGP Configuration (IPv4 and IPv6)

```cisco
router bgp 65000
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.2 remote-as 65001
 neighbor 2001:db8:acad:1::2 remote-as 65000
 
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  network 192.168.10.0 mask 255.255.255.0
 exit-address-family

 address-family ipv6 unicast
  neighbor 2001:db8:acad:1::2 activate
  neighbor 2001:db8:acad:1::2 next-hop-self
  network 2001:db8:cafe:1::/64
 exit-address-family
```

**Command Breakdown & Explanation:**

* `router bgp 65000`: Initializes the BGP process with the local ASN of 65000.
* `neighbor ... remote-as`: Defines the neighbor and its ASN. `10.0.0.2` is an eBGP peer (different ASN), while `2001:db8:acad:1::2` is an iBGP peer (same ASN).
* `address-family ... unicast`: Activates the specific address family (IPv4 or IPv6) for multiprotocol routing.
* `neighbor ... activate`: Explicitly enables the exchange of routes for the defined address family.
* `neighbor ... next-hop-self`: Used primarily in iBGP. By default, eBGP routes advertised into iBGP retain their original next-hop IP. This command forces the local router to change the next-hop attribute to itself, ensuring internal peers can reach the destination.
* `network ... mask`: In BGP, the `network` command does not enable BGP on an interface. Instead, it instructs BGP to look in the global routing table for an *exact match* of the specified subnet and mask. If it exists, BGP advertises it.

## 2. BGP Path Selection (Best Path Algorithm)

BGP does not use a simple metric like bandwidth or delay. Instead, it evaluates a sequence of Path Attributes to determine the best path to a destination. The router evaluates these top-down; the first attribute that breaks the tie is chosen.

| Attribute | Scope | Default | Description |
| :--*| :--* | :--*| :--* |
| **Weight** | Cisco Proprietary, Local Router | 0 (32768 for local) | Highest weight wins. It is never advertised to any neighbor. |
| **Local Preference** | Entire Local AS (iBGP) | 100 | Highest Local Pref wins. Used to determine the exit point *out* of your AS. |
| **Locally Originated** | Local Router | N/A | Prefer routes originated locally (via network or aggregate commands). |
| **AS_Path Length** | Global | N/A | Shortest AS_Path list wins. Can be manipulated via AS prepending. |
| **Origin Code** | Global | N/A | Prefers IGP (`i`) over EGP (`e`) over Incomplete (`?`). |
| **MED (Multi-Exit Discriminator)** | Neighbor AS | 0 | Lowest MED wins. Used to tell an external AS how to enter your AS. |
| **eBGP over iBGP** | Local Router | N/A | Prefers external paths over internal paths. |

### 2.1 Manipulating Path Selection (Local Preference & Weight)

```cisco
route-map PREFER-ISP1 permit 10
 set local-preference 200
 exit

router bgp 65000
 address-family ipv4 unicast
  neighbor 10.0.0.2 route-map PREFER-ISP1 in
 exit-address-family
```

**Command Breakdown & Explanation:**

* `set local-preference 200`: Modifies the Local Preference attribute to 200 (higher than the default 100).
* `neighbor ... route-map ... in`: Applies the route-map to incoming updates from ISP1. Because Local Preference is shared across the entire iBGP domain, this configuration ensures all internal routers will prefer this exit path for the learned routes.

## 3. Advanced BGP Features

```markdown
This section covers scalability and administrative control features, including Route Summarization, BGP Communities, and Peer Templates.
```

### 3.1 Route Summarization (Aggregation)

```cisco
router bgp 65000
 address-family ipv4 unicast
  aggregate-address 172.16.0.0 255.255.0.0 summary-only as-set
 exit-address-family
```

**Command Breakdown & Explanation:**

* `aggregate-address`: Creates a summary route in the BGP table. For this to work, at least one subset of this aggregate must exist in the BGP routing table.
* `summary-only`: Suppresses the advertisement of the granular subnets, advertising *only* the summarized route.
* `as-set`: Preserves the AS_Path history of all the suppressed granular routes within the aggregate. This prevents routing loops when summarizing across multiple AS domains.

### 3.2 BGP Communities and Peer Groups

```cisco
router bgp 65000
 template peer-policy MY-IBGP-POLICY
  send-community both
  next-hop-self
 exit-peer-policy

 address-family ipv4 unicast
  neighbor 192.168.1.2 inherit peer-policy MY-IBGP-POLICY
  neighbor 192.168.1.3 inherit peer-policy MY-IBGP-POLICY
 exit-address-family
```

**Command Breakdown & Explanation:**

* `template peer-policy`: Creates a reusable template (replacing legacy peer groups in modern IOS XE) to apply common configurations to multiple neighbors, reducing CPU load and configuration length.
* `send-community both`: By default, BGP strips community tags. This instructs BGP to forward both Standard and Extended communities to the peer. Communities are tags attached to routes that dictate routing policies (e.g., `no-export` prevents a route from being advertised to an eBGP peer).
* `inherit peer-policy`: Binds the neighbor to the defined template.

## 4. Troubleshooting and Verification

### 4.1 Verify BGP Session Status

Displays a high-level summary of all BGP neighbors, their ASN, and their connection state.

**Command:** `show bgp ipv4 unicast summary`

**Command Breakdown & Explanation:**
Validates whether TCP connections have been successfully established with configured peers and whether prefixes are being actively received.

What it checks and variables to look for:

* **Neighbor**: Must list the correct peer IP.
* **V (Version)**: Must be `4`.
* **AS**: Must match the expected remote ASN.
* **State/PfxRcd**: Must display a number (indicating the number of prefixes received). If it says `Active`, `Idle`, or `Connect`, the TCP session is down or failing.

### 4.2 Verify BGP Routing Table

Displays the full BGP routing table, including next-hops, metrics, and path attributes for all learned prefixes.

**Command:** `show bgp ipv4 unicast`

**Command Breakdown & Explanation:**
Examines the BGP topology to determine which paths have been selected as the best path and verifies that attributes like Local Preference or AS_Path are correctly applied.

What it checks and variables to look for:

* **Network**: Must list the expected subnet.
* **Next Hop**: Must be a reachable IP address. (If it shows `0.0.0.0`, the route is locally originated).
* **Status Codes**: The line must begin with `*>`. The `*` means the route is valid (next-hop is reachable), and the `>` means BGP has selected it as the best path.
* **Path**: Displays the AS_Path attribute.

### 4.3 Verify BGP Neighbor Details

Provides an extensive, detailed look into a specific BGP neighbor adjacency, including hold times, supported capabilities, and applied route-maps.

**Command:** `show ip bgp neighbors 10.0.0.2`

**Command Breakdown & Explanation:**
Crucial for deep-dive troubleshooting of session drops, capability mismatches (like multiprotocol support), or verifying that filtering policies are actively attached to the peer.

What it checks and variables to look for:

* **BGP state**: Must be `Established`.
* **Neighbor capabilities**: Should indicate `Multiprotocol IPv4 Unicast` or `IPv6 Unicast` as `advertised and received`.
* **Message statistics**: Ensures `Keepalives` are actively incrementing.
* **Connections established / dropped**: A high drop count indicates link flapping or MTU/TCP issues.

<!-- Created by: Gergő Téringer, 2026 -->