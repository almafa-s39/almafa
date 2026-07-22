<!-- 
---
title: "EIGRP Comprehensive Configuration and Advanced Features"
author: "Gergő Téringer"
---
 -->
# EIGRP Comprehensive Configuration and Advanced Features

```markdown
This document provides a comprehensive configuration, operations, and troubleshooting reference for the Enhanced Interior Gateway Routing Protocol (EIGRP). It merges foundational configurations (Classic and Named Modes, IPv4/IPv6) with advanced features (Stub routing, summarization, metric manipulation, and load balancing).

> [!IMPORTANT]
> Modern Context & Compatibility: In modern enterprise networks (e.g., Cisco IOS-XE Catalyst 8000 series), EIGRP Named Mode is the recommended standard. It consolidates IPv4 and IPv6 configurations under a single router process and natively supports Next-Generation Cryptography, such as SHA-256 authentication. This ensures routing protocol security meets compliance standards required when transporting sensitive traffic, such as Windows Server 2025 Active Directory replication or modern hybrid-cloud data plane traffic.

> [!NOTE]
> Ensure that `auto-summary` is explicitly disabled (or verify it is disabled by default in modern IOS versions) to ensure that branch router subnets are advertised exactly as configured (e.g., as a `/24`), rather than being summarized to their classful network boundaries (e.g., `/22` or `/16`).
```

## 1. EIGRP Configuration Modes

Cisco IOS supports two methods for configuring EIGRP: the legacy Classic Mode (separated by IPv4 and IPv6) and the modern Named Mode (Multi-Agent mode), which integrates all address families into a unified hierarchical structure.

### 1.1 Classic Configuration Mode

```cisco
router eigrp 100
 eigrp router-id 1.1.1.1
 network 10.0.0.0 0.255.255.255
 network 192.168.10.0 0.0.0.255
 no auto-summary
 passive-interface default
 no passive-interface GigabitEthernet0/0
```

**Command Breakdown & Explanation:**

- `router eigrp 100`: Initializes the EIGRP process for Autonomous System (AS) 100.
- `eigrp router-id`: Manually assigns a 32-bit unique identifier to the router, preventing neighbor flapping if physical interface IPs change.
- `network ...`: Enables EIGRP on interfaces matching the specified IP address and wildcard mask, adding those networks to the EIGRP topology table.
- `no auto-summary`: Prevents the router from summarizing subnets to their classful boundary when crossing network boundaries, preserving granular `/24` branch subnets.
- `passive-interface default`: Secures the routing process by preventing EIGRP Hello packets from being sent out any interface by default.
- `no passive-interface`: Explicitly allows EIGRP adjacencies to form on designated uplink/transit interfaces.

### 1.2 EIGRP Named Mode

```cisco
router eigrp CORE-ROUTING
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
  network 10.0.0.0 0.255.255.255
  network 192.168.10.0 0.0.0.255
  
  af-interface default
   passive-interface
  exit-af-interface
  
  af-interface GigabitEthernet0/0
   no passive-interface
   authentication mode hmac-sha-256 Passw0rd123!
  exit-af-interface
 exit-address-family
```

**Command Breakdown & Explanation:**

- `router eigrp CORE-ROUTING`: Initializes Named Mode using an arbitrary local string.
- `address-family ipv4 ...`: Enters the IPv4 topology for AS 100.
- `af-interface default` / `passive-interface`: Applies passive interface status globally within the address family.
- `authentication mode hmac-sha-256`: Secures the neighbor adjacency on `GigabitEthernet0/0` using modern SHA-256 cryptographic hashing.

## 2. EIGRPv6 Configuration

EIGRPv6 functions nearly identically to EIGRP for IPv4, utilizing the same DUAL algorithm, but operates over IPv6 link-local addresses for neighbor discovery and updates.

### 2.1 Enabling EIGRPv6 (Named Mode)

```cisco
router eigrp CORE-ROUTING
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
  
  af-interface default
   passive-interface
  exit-af-interface
  
  af-interface GigabitEthernet0/0
   no passive-interface
  exit-af-interface
 exit-address-family
```

**Command Breakdown & Explanation:**

- `address-family ipv6 unicast`: Initializes the EIGRPv6 process. Note that EIGRPv6 requires a 32-bit IPv4-formatted `router-id` to function; if no IPv4 addresses exist on the router, it must be manually defined, or the process will fail to start. Interfaces in Named Mode are automatically enabled for EIGRPv6 if they have an IPv6 address configured.

## 3. EIGRP Advanced Features and Operations

This section details advanced operational mechanics, including EIGRP Stub routing, metric calculations, summarization, and load balancing mechanisms.

> [!TIP]
> Strictly controlling EIGRP query boundaries using Stub routing and route summarization prevents unnecessary DUAL calculations across the WAN. This prevents "Stuck In Active" (SIA) events and ensures rapid convergence for real-time applications.

### 3.1 EIGRP Stub Routing

EIGRP Stub routing is primarily used in Hub-and-Spoke topologies to improve network stability. When a router is configured as a stub, it immediately informs its neighbors, which will then exclude the stub router from any queries regarding lost routes.

| Stub Parameter | Operational Behavior |
| :--- | :--- |
| **connected** | Advertises directly connected networks that are matched by a `network` statement. |
| **summary** | Advertises automatically or manually summarized routes. (This is active by default alongside `connected`). |
| **static** | Advertises static routes that have been explicitly redistributed into the EIGRP process. |
| **receive-only** | Prevents the stub router from advertising *any* routes to its neighbors. It only receives routing updates. |
| **redistributed** | Advertises external routes that have been redistributed into EIGRP from other routing protocols. |
| **leak-map** | Allows a stub router to advertise specific, granular prefixes that would otherwise be restricted by the stub configuration, using a route-map. |

```cisco
router eigrp CORE-ROUTING
 address-family ipv4 unicast autonomous-system 100
  topology base
   eigrp stub connected summary
  exit-af-topology
 exit-address-family
```

**Command Breakdown & Explanation:**

- `topology base` / `eigrp stub connected summary`: Configures the router as an EIGRP stub within Named Mode. It tells the router to only advertise connected subnets and any configured summary routes to the upstream Hub, dropping all other transit routing advertisements.

### 3.2 Advanced Route Summarization

Summarization reduces the size of the EIGRP topology table, limits the query boundary during network convergence, and conserves router memory and CPU.

```cisco
router eigrp CORE-ROUTING
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/0
   summary-address 172.16.0.0 255.255.248.0
  exit-af-interface
 exit-address-family
```

**Command Breakdown & Explanation:**

- `summary-address`: Advertises a single summary route (`172.16.0.0/21`) out of `GigabitEthernet0/0` instead of all the individual subnets that fall within that range. A summary discard route (Null0) with an Administrative Distance of 5 is automatically generated locally to prevent routing loops.

### 3.3 EIGRP Metric and K-Values

EIGRP uses a composite metric calculated using the Diffusing Update Algorithm (DUAL). To calculate the optimal path, EIGRP considers multiple attributes of a link, which are weighted using constant values known as "K-Values".

> [!WARNING]
> By default, EIGRP only uses Bandwidth and Delay to calculate its metric. K-Values MUST match between two routers for an EIGRP neighbor adjacency to form.

| K-Value | Metric Component | Default State | Description |
| :--- | :--- | :--- | :--- |
| **K1** | Bandwidth | **Enabled (1)** | The lowest bandwidth along the path to the destination. |
| **K2** | Load | Disabled (0) | The worst load on a link along the path (dynamically changes). |
| **K3** | Delay | **Enabled (1)** | The cumulative sum of all interface delays along the path to the destination. |
| **K4** | Reliability | Disabled (0) | The worst reliability along the path based on keepalives/errors. |
| **K5** | MTU | Disabled (0) | Maximum Transmission Unit. |
| **K6** | Extended Metrics | Disabled (0) | Used in EIGRP Named Mode "Wide Metrics" to support high-speed interfaces. |

```cisco
interface GigabitEthernet0/1
 delay 1000
```

**Command Breakdown & Explanation:**

- `delay 1000`: Increases the logical delay of the interface (measured in tens of microseconds). Because manipulating bandwidth can break QoS policies, modifying delay is the industry standard method to artificially increase an EIGRP path metric, making it a less desirable backup path.

### 3.4 Unequal Cost Load Balancing (Variance)

EIGRP can load balance across multiple links with *different* metrics using the `variance` multiplier. For a backup route (Feasible Successor) to be installed into the routing table alongside the best route (Successor), its metric must be less than the best route's metric multiplied by the variance value.

```cisco
router eigrp CORE-ROUTING
 address-family ipv4 unicast autonomous-system 100
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
 exit-address-family
```

**Command Breakdown & Explanation:**

- `variance 2`: Instructs EIGRP to install any valid Feasible Successor route into the global routing table if its metric is less than 2 times the metric of the primary Successor route.
- `maximum-paths 4`: Defines the maximum number of parallel routes EIGRP can install in the routing table for a single destination.

## 4 Troubleshooting and Verification

### 4.1 Verify EIGRP Neighbor Adjacencies

Validates that local routers have successfully formed peering relationships and are exchanging hello packets.

**Command:** `show ip eigrp neighbors`

**Command Breakdown & Explanation:**
Checks the status of neighboring EIGRP routers. A failure here often indicates mismatched Autonomous System numbers, mismatched K-values, or an interface being passively suppressed.

What it checks and variables to look for:

- **Address**: Must list the IP of the remote neighbor
- **Interface**: Must list the correct physical or logical interface connecting to the peer
- **Q Cnt**: Must be `0`
- **State/Uptime**: Should show an established time format without constantly resetting

### 4.2 Verify EIGRP Topology and Convergence

Examines the EIGRP topology table to ensure routes are stable and the DUAL algorithm is not actively recalculating paths.

**Command:** `show ip eigrp topology`

**Command Breakdown & Explanation:**
Displays all Successor (primary) and Feasible Successor (backup) routes. Crucial for diagnosing "Stuck In Active" (SIA) failures.

What it checks and variables to look for:

- **State**: Must be `P` (Passive, indicating a stable route)
- **Successors**: Must be `>= 1` for the route to be injected into the routing table
- **FD (Feasible Distance)**: Must be a calculated metric number

### 4.3 Verify EIGRP Protocol Settings and K-Values

Displays the global parameters of the active EIGRP routing process, including K-Values, variance multipliers, and router IDs.

**Command:** `show ip protocols`

**Command Breakdown & Explanation:**
Validates the administrative configurations of the EIGRP protocol. This is the fastest way to check if K-Values match standard defaults or if a variance is actively applied.

What it checks and variables to look for:

- **Routing Protocol**: Must be `eigrp 100` (or your chosen AS)
- **Metric weight**: Must show the K-values, typically `K1=1, K2=0, K3=1, K4=0, K5=0`
- **Variance**: Should indicate your configured multiplier (e.g., `2`)

### 4.4 Verify EIGRP Neighbor Stub Information

Provides a granular look at the established EIGRP adjacencies, specifically detailing the features supported by the remote neighbor, such as their Stub status.

**Command:** `show ip eigrp neighbors detail`

**Command Breakdown & Explanation:**
Validates whether a remote branch router has successfully advertised its stub status to the local Hub router, confirming that queries will be suppressed.

What it checks and variables to look for:

- **Stub Peer**: Must state `yes` if the remote router is configured as a stub
- **Stub Options**: Must reflect the configured parameters (e.g., `connected summary`)
- **Suppress**: Should indicate `yes` for suppressed queries

### 4.5 Verify EIGRPv6 Interfaces

Validates which IPv6-enabled interfaces are actively participating in the EIGRPv6 process.

**Command:** `show ipv6 eigrp interfaces`

**Command Breakdown & Explanation:**
Ensures that the EIGRP routing process has successfully attached to the correct IPv6 interfaces and that peers are being discovered on those links.

What it checks and variables to look for:

- **Interface**: Must list the expected interfaces (e.g., `Gi0/0`)
- **Peers**: Must be `>= 1` on transit links where a neighbor is expected
- **Xmit Queue**: Must be `0/0` (indicating no stuck packets)

<!-- Created by: Gergő Téringer, 2026 -->