<!-- 
---
title: "OSPFv2"
author: "Gergő Téringer"
---
-->
# OSPFv2

Open Shortest Path First (OSPF) version 2 is a robust, link-state interior gateway protocol designed for IPv4. Building a comprehensive reference library with these advanced OSPFv2 configurations perfectly aligns with the depth required for an ENARSI-level engineering environment.

The following documentation covers everything from fundamental neighbor adjacencies to advanced stub area designs and summarization

## 1. OSPF Fundamentals & Adjacencies

The foundation of OSPF involves establishing the router ID, defining which interfaces participate in the routing process, and securing the edge of the network using passive interfaces to prevent accidental neighbor formations.

```cisco
router ospf 1
 router-id 10.255.255.1
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.10.10.0 0.0.0.255 area 0

interface GigabitEthernet0/1
 ip ospf 1 area 1
```

**Command Breakdown & Explanation:**

- `router ospf 1`: Initializes the OSPF process locally with a process ID of `1`.
- `router-id 10.255.255.1`: Statically defines the router ID. If not defined, OSPF chooses the highest loopback IP, or the highest active physical IP. A static ID provides stability.
- `passive-interface default`: Secures the router by stopping OSPF Hello packets from being sent out of any interface by default.
- `no passive-interface ...`: Explicitly permits Hello packets on specific transit links.
- `network 10.10.10.0 0.0.0.255 area 0`: The legacy method of enabling OSPF. It uses a wildcard mask to match interface IP addresses and assign them to `area 0`.
- `ip ospf 1 area 1`: The modern, interface-specific configuration method. It directly enables OSPF process 1 and assigns the interface to `area 1`, overriding any global `network` statements.

## 2. Default Route Advertisement & External Routes

OSPF can inject external routes into the domain via redistribution, or dynamically generate a default route to direct outbound traffic toward an ISP edge router.

```cisco
router ospf 1
 default-information originate always metric 10 metric-type 1
 redistribute static subnets route-map RM-STATIC-TO-OSPF
```

**Command Breakdown & Explanation:**

- `default-information originate always`: Injects a default route `0.0.0.0/0` into OSPF. The `always` keyword forces the router to advertise it even if it does not have a default route in its own routing table.
- `metric-type 1`: Changes the default External Type 2 (E2) route to an External Type 1 (E1) route. E1 routes accurately calculate the total internal path cost to reach the ASBR, whereas E2 routes only display the static external cost.
- `redistribute static subnets`: Injects static routes into OSPF. The `subnets` keyword is critical; without it, OSPF will only redistribute classful networks.

## 3. Network Types, DR/BDR, and Timers

OSPF behaves differently depending on the Layer 2 topology. Modifying network types and timers ensures rapid failure detection and optimizes Designated Router (DR) elections.

| OSPF Network Type | DR/BDR Election | Timers (Hello/Dead) | Traffic Type | Default Interface / Use Case |
| :--- | :--- | :--- | :--- | :--- |
| `Broadcast` | Yes | 10 / 40 seconds | Multicast | Ethernet (LAN). Requires full mesh. |
| `Non-Broadcast` | Yes | 30 / 120 seconds | Unicast | Legacy Frame Relay / Hub-and-Spoke. |
| `Point-to-Point` | No | 10 / 40 seconds | Multicast | Serial links or direct router-to-router Ethernet. |
| `Point-to-Multipoint` | No | 30 / 120 seconds | Multicast | Partial mesh / Hub-and-Spoke. Adds /32 host routes. |
| `Loopback` | No | N/A | N/A | Loopback interfaces. Advertised as /32. |

**Configuration:**

```cisco
interface GigabitEthernet0/0
 description TRANSIT-BROADCAST
 ip ospf priority 255
 ip ospf hello-interval 2
 ip ospf dead-interval 8

interface GigabitEthernet0/1
 description P2P-LINK
 ip ospf network point-to-point
```

**Command Breakdown & Explanation:**

- `ip ospf priority 255`: Guarantees this router will win the DR election for this broadcast segment, as `255` is the highest priority. A priority of `0` ensures a router never becomes the DR or BDR.
- `ip ospf hello-interval 2` and `dead-interval 8`: Aggressively lowers the failure detection timers from the default `10/40` seconds to `2/8` seconds. Timers must match exactly between neighbors for an adjacency to form.
- `ip ospf network point-to-point`: Optimizes a direct router-to-router link by disabling the unnecessary DR/BDR election process, speeding up convergence.

## 4. OSPF Authentication

Securing OSPF prevents rogue devices from injecting false routing information into the domain. Authentication can be enabled per-interface or globally for an entire area.

**Configuration:**

```cisco
! Area-Wide Authentication
router ospf 1
 area 0 authentication message-digest

! Interface-Specific Key Configuration
interface GigabitEthernet0/0
 ip ospf message-digest-key 1 md5 Passw0rd!

! Interface-Specific Authentication (Overrides Area setting if used)
interface GigabitEthernet0/1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Passw0rd!
```

**Command Breakdown & Explanation:**

- `area 0 authentication message-digest`: Globally instructs the router to enforce MD5 authentication for all interfaces participating in Area 0.
- `ip ospf authentication message-digest`: Enables MD5 authentication specifically on a single interface.
- `ip ospf message-digest-key 1 md5 ...`: Defines the key ID and the shared secret hash. These must match exactly on the neighboring router, regardless of whether you used the interface or area command to enable the feature.

## 5. OSPF Link-State Advertisements (LSAs)

OSPF builds its topology database (LSDB) by exchanging Link-State Advertisements. Understanding LSAs is critical for troubleshooting advanced routing and summarization issues.

| LSA Type | Name | Generated By | Description / Scope |
| :--- | :--- | :--- | :--- |
| `Type 1` | Router LSA | Every Router | Describes the router's attached interfaces and states. Flooded only within the local area. |
| `Type 2` | Network LSA | DR | Describes all routers attached to a broadcast segment. Flooded only within the local area. |
| `Type 3` | Summary LSA | ABR | Summarizes intra-area routes (Type 1/2) and injects them into a neighboring area. |
| `Type 4` | ASBR Summary | ABR | Provides the route/cost to reach an ASBR. Required for Type 5 LSAs to resolve their next-hop. |
| `Type 5` | External LSA | ASBR | Describes routes redistributed into OSPF from external sources (e.g., BGP, EIGRP, Static). Flooded domain-wide. |
| `Type 7` | NSSA External | ASBR in an NSSA | External routes injected within a Not-So-Stubby Area. The ABR converts Type 7 to Type 5 before flooding to Area 0. |

## 6. Advanced OSPF: Stubby Areas

Stub areas optimize router resources by filtering out specific LSA types from entering an area, replacing them with a default route to maintain reachability.

| Area Type | Blocks LSA 4 & 5 (External)? | Blocks LSA 3 (Inter-Area)? | Allows LSA 7 (ASBR)? | Auto-Injects Default Route? |
| :--- | :--- | :--- | :--- | :--- |
| `Stub` | Yes | No | No | Yes (Type 3 Default) |
| `Totally Stubby` | Yes | Yes | No | Yes (Type 3 Default) |
| `NSSA` | Yes | No | Yes | No (Must configure manually) |
| `Totally NSSA` | Yes | Yes | Yes | Yes (Type 3 Default) |

**Configuration:**

```cisco
! On Area Border Router (ABR)
router ospf 1
 area 10 stub
 area 20 stub no-summary
 area 30 nssa default-information-originate
 area 40 nssa no-summary
```

**Command Breakdown & Explanation:**

- `area 10 stub`: Configures Area 10 as a standard Stub Area.
- `area 20 stub no-summary`: Configures Area 20 as a Totally Stubby Area. The `no-summary` keyword blocks the Type 3 LSAs.
- `area 30 nssa default-information-originate`: Configures a Not-So-Stubby Area. Because NSSAs do not inject a default route automatically on Cisco IOS, the `default-information-originate` command is required to maintain outbound connectivity.
- `area 40 nssa no-summary`: Configures a Totally NSSA. The `no-summary` keyword automatically triggers the injection of a default route, so the manual originate command is not needed here.

## 7. Route Summarization & Path Selection

Summarization reduces the size of the routing table and limits the scope of topology changes (LSA flooding). Path selection is manipulated by altering interface costs.

**Configuration:**

```cisco
router ospf 1
 auto-cost reference-bandwidth 100000
 area 1 range 10.1.0.0 255.255.0.0
 summary-address 172.16.0.0 255.255.252.0

interface GigabitEthernet0/0
 ip ospf cost 500
```

**Command Breakdown & Explanation:**

- `auto-cost reference-bandwidth 100000`: By default, OSPF treats a 100Mbps link and a 10Gbps link as having the same cost (`1`). Setting this to `100,000` Megabits adjusts the formula so high-speed links are correctly preferred. This must be set on all routers in the domain.
- `area 1 range ...`: Configures Inter-area (Type 3 LSA) summarization on an ABR. All subnets within `10.1.X.X` in Area 1 are summarized into a single route advertised to Area 0.
- `summary-address ...`: Configures External (Type 5 LSA) summarization on an ASBR before the routes are injected into OSPF.
- `ip ospf cost 500`: Manually manipulates the metric of an interface to influence path selection without changing the physical bandwidth.

## 8. Virtual Links

A Virtual Link is a temporary fix used when an OSPF area cannot physically connect directly to Area 0 (the backbone). It tunnels Area 0 traffic through a transit area.

**Configuration:**

```cisco
! Configured on the ABR connecting to Area 0, and the isolated ABR
router ospf 1
 area 1 virtual-link 10.255.255.4
```

**Command Breakdown & Explanation:**

- `area 1 virtual-link 10.255.255.4`: Placed under the OSPF process. `Area 1` is the transit area that both routers share. `10.255.255.4` is the OSPF Router ID of the neighbor on the other side of the transit area, establishing the logical connection.

## 9. Verification and troubleshooting

Verifying OSPF involves checking the operational state of neighbors, interfaces, the routing table, and the link-state database. The following commands are essential for troubleshooting adjacency issues and routing loops.

### 9.1 Verifying Neighbor Adjacencies

This command is the first step in troubleshooting. If routers do not form a full adjacency, they will not exchange routing information.

**Command:** `show ip ospf neighbor`

**What it checks and variables to look for:**

- `Neighbor ID`: The OSPF Router ID of the connected device. If this is missing, OSPF is not receiving Hello packets.
- `State`: The current adjacency state. You want to see `FULL` for a complete adjacency. On broadcast networks, `FULL/DR` or `FULL/BDR` is expected, while `2WAY/DROTHER` is normal between two non-designated routers. Stuck states like `EXSTART` or `EXCHANGE` usually indicate an MTU mismatch.
- `Dead Time`: A countdown timer. If this reaches 0, the neighbor is declared dead. It should constantly reset when Hello packets are received.
- `Interface`: The local physical interface where this neighbor is connected.

### 9.2 Verifying Interface Parameters

This command provides a quick overview of all interfaces actively participating in the OSPF process, which is useful for confirming your `network` or interface-level configurations were applied correctly.

**Command:** `show ip ospf interface brief`

**What it checks and variables to look for:**

- `PID`: The local OSPF Process ID running on that interface.
- `Area`: The OSPF Area assigned to the interface. A mismatch here will prevent adjacency.
- `Cost`: The OSPF metric assigned to the link. Useful for verifying path selection manipulation.
- `State`: Shows the interface's role on the segment (e.g., `DR`, `BDR`, `P2P`, or `LOOP`).
- `Nbrs F/C`: The ratio of Fully adjacent neighbors to Connected neighbors on that specific link.

### 9.3 Verifying the Routing Table

Once adjacencies are formed, you need to verify that the router is actually installing the expected OSPF routes into its global routing table.

**Command:** `show ip route ospf`

**What it checks and variables to look for:**

- `Route Codes`: Identifies the type of OSPF route. `O` means an intra-area route (Type 1/2 LSA). `O IA` means an inter-area route (Type 3 LSA). `O E1` or `O E2` indicate external routes (Type 5 LSA), and `O N1` or `O N2` indicate NSSA external routes (Type 7 LSA).
- `Subnet`: The destination network.
- `[110/Cost]`: The Administrative Distance of OSPF is `110`. The second number is the total cumulative OSPF cost to reach that subnet.
- `Via`: The next-hop IP address to forward traffic toward that destination.

### 9.4 Verifying the Link-State Database (LSDB)

If routes are missing from the routing table, you must check the database to see if the router actually received the LSA. If the LSA is in the database but not the routing table, there is usually a topology or filtering issue.

**Command:** `show ip ospf database`

**What it checks and variables to look for:**

- `Link ID`: The identifier for the LSA. For a Router LSA, this is the Router ID. For a Network LSA, it is the DR's interface IP. For a Summary or External LSA, it is the actual subnet address.
- `ADV Router`: The OSPF Router ID of the device that originally created and flooded this LSA.
- `Age`: How old the LSA is in seconds. OSPF refreshes LSAs every 1800 seconds (30 minutes). If an LSA reaches 3600 seconds (MaxAge), it is flushed from the database.
- `Seq#`: The sequence number. Every time an LSA is updated, this number increments, allowing routers to easily determine which LSA information is the most recent.

<!-- Created by: Gergő Téringer, 2026 -->