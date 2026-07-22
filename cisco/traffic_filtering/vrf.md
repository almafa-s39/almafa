<!-- 
---
title: "Virtual Routing and Forwarding (VRF) in Routing Protocols"
author: "Gergő Téringer"
---
 -->
# Virtual Routing and Forwarding (VRF) in Routing Protocols

Virtual Routing and Forwarding (VRF) is a technology that allows multiple instances of a routing table to co-exist within the same router at the same time. Because the routing instances are independent, the same or overlapping IP addresses can be used without conflicting with each other. This documentation covers how to define VRFs and integrate them into the three primary interior and exterior gateway protocols: OSPF, EIGRP, and BGP.

> [!IMPORTANT]
> Modern Context & Compatibility: In contemporary enterprise architectures, VRF-Lite is heavily utilized to support multi-tenancy. For example, if you are isolating Guest Wi-Fi, IoT devices, or segregating management traffic for modern Windows Server 2025 Active Directory environments, VRFs ensure that broadcast domains and routing tables remain strictly segmented across the transport core. This maps directly to hypervisor-level segmentation (e.g., Windows Server Hyper-V Network Virtualization or VMware NSX) down to the physical underlay switches.

## 1. Global VRF Definition

Before assigning a routing protocol to a VRF, the VRF instance must be globally defined on the router and an interface must be bound to it.

> [!WARNING]
> Assigning an interface to a VRF will instantly remove any currently configured IP addresses on that interface. You must reapply the IP address after the `vrf forwarding` command is issued.

```cisco
vrf definition TENANT-A
 rd 65000:10
 route-target export 65000:10
 route-target import 65000:10
 address-family ipv4
 exit-address-family
 exit

interface GigabitEthernet0/1
 vrf forwarding TENANT-A
 ip address 10.10.10.1 255.255.255.0
 no shutdown
```

**Command Breakdown & Explanation:**

- `vrf definition`: Creates a new VRF instance named `TENANT-A`.
- `rd`: Defines the Route Distinguisher. This value (e.g., `65000:10`) is prepended to the IPv4 prefix to create a unique 96-bit VPNv4 address, which is critical when BGP exchanges routes between VRFs.
- `route-target`: Specifies the extended BGP communities used to import and export routes into and out of this specific VRF.
- `address-family ipv4`: Initializes the IPv4 routing table for this VRF.
- `vrf forwarding`: Binds the physical interface to the VRF context.

## 2. VRF Integration with OSPF

To run OSPF within a VRF, you must define a unique OSPF process ID and explicitly bind it to the VRF. The router will maintain a separate Link-State Database (LSDB) specifically for this instance.

```cisco
router ospf 10 vrf TENANT-A
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.255 area 0
 capability vrf-lite
```

**Command Breakdown & Explanation:**

- `router ospf 10 vrf TENANT-A`: Starts OSPF process `10` and explicitly ties it to the `TENANT-A` routing table.
- `router-id`: Sets the OSPF router ID for this specific VRF process. It is highly recommended to manually set this, as the router might not find a valid loopback interface inside the VRF to pick automatically.
- `capability vrf-lite`: Disables the OSPF Provider Edge (PE) specific checks (like the Down bit and domain-tag checks). This is required when running VRF-Lite between customer edge routers without an MPLS backbone, preventing valid LSAs from being discarded.

## 3. VRF Integration with EIGRP

EIGRP configuration for VRFs is handled under "Named Mode" (also known as Multi-Agent mode). You define a global EIGRP virtual instance and then place the VRF under a specific IPv4 address family and Autonomous System (AS).

> [!TIP]
> When dealing with EIGRP in a VRF, remember that `auto-summary` is disabled by default in modern IOS-XE versions. This ensures that branch router subnets are advertised exactly as they are configured (e.g., as a `/24` rather than summarized to a classful `/22` or `/16` boundary), preserving granular routing within the tenant instance.

```cisco
router eigrp MULTI-TENANT
 address-family ipv4 vrf TENANT-A autonomous-system 100
  network 10.10.10.0 0.0.0.255
  eigrp router-id 2.2.2.2
  exit-address-family
```

**Command Breakdown & Explanation:**

- `router eigrp MULTI-TENANT`: Initializes the EIGRP Named Mode routing process using an arbitrary virtual name (`MULTI-TENANT`).
- `address-family ipv4 vrf`: Binds the IPv4 address family for `TENANT-A` to the EIGRP process and assigns it the operational Autonomous System number of `100`.
- `eigrp router-id`: Assigns a unique identifier to the EIGRP instance operating inside this specific VRF.

## 4. VRF Integration with BGP

BGP handles VRFs uniquely through the MP-BGP (Multiprotocol BGP) extension. Neighbors are defined under the global BGP process but activated specifically within the VRF address family.

```cisco
router bgp 65000
 bgp log-neighbor-changes
 address-family ipv4 vrf TENANT-A
  neighbor 10.10.10.2 remote-as 65001
  neighbor 10.10.10.2 activate
  network 192.168.100.0 mask 255.255.255.0
  exit-address-family
```

**Command Breakdown & Explanation:**

- `router bgp 65000`: Starts the global BGP process using the local AS number `65000`.
- `address-family ipv4 vrf TENANT-A`: Enters the BGP address family configuration specifically for the `TENANT-A` VRF.
- `neighbor ... remote-as`: Defines the peering IP address and the remote AS number. This IP must be reachable via an interface bound to `TENANT-A`.
- `neighbor ... activate`: Explicitly enables the exchange of IPv4 routes with the neighbor for this specific VRF.
- `network ... mask`: Advertises a specific prefix from the `TENANT-A` routing table into the BGP process.

## 5. Troubleshooting and Verification

### 5.1 Verify VRF Interfaces

Displays all active VRF instances and the physical or logical interfaces associated with them.

**Command:** `show vrf interfaces`

**Command Breakdown & Explanation:**
Validates that your physical ports or subinterfaces have been successfully bound to the correct VRF instances.

What it checks and variables to look for:

- **Interface**: Must list the correct physical interface (e.g., `Gi0/1`).
- **VRF**: Must match your tenant string (e.g., `TENANT-A`).
- **Status**: Must be `up`.

### 5.2 Verify OSPF VRF Neighbors

Checks the OSPF adjacency status specifically for a given VRF process.

**Command:** `show ip ospf neighbor vrf TENANT-A`

**Command Breakdown & Explanation:**
Validates that OSPF Hello packets are successfully passing across the VRF-assigned interfaces and that the Link-State databases are synchronized.

What it checks and variables to look for:

- **State**: Must be `FULL` (or `2WAY` if on a Drother multi-access segment).
- **Interface**: Must be an interface belonging to the VRF.

### 5.3 Verify EIGRP VRF Topology

Displays the EIGRP topology table for the specified VRF, showing successors and feasible successors.

**Command:** `show ip eigrp vrf TENANT-A topology`

**Command Breakdown & Explanation:**
Verifies that EIGRP is learning remote subnets within the VRF boundary and calculating the correct metric paths.

What it checks and variables to look for:

- **State**: Must be `P` (Passive, which means the route is stable and not actively seeking a path).
- **Successors**: Must be `>= 1` for a valid route to be injected into the routing table.

### 5.4 Verify BGP VRF Routing Table

Displays the routing entries learned and advertised via BGP inside the isolated VRF instance.

**Command:** `show bgp vrf TENANT-A ipv4 unicast`

**Command Breakdown & Explanation:**
Checks the Multi-Protocol BGP table to ensure prefixes are being received from neighbors and marked as valid and best paths.

What it checks and variables to look for:

- **Network**: Must list the remote subnets expected from the peer.
- **Next Hop**: Must show the correct neighbor IP address (e.g., `10.10.10.2`).
- **Status Codes**: The prefix line must begin with `*>`, indicating the route is both valid (`*`) and selected as the best path (`>`).
<!-- Created by: Gergő Téringer, 2026 -->