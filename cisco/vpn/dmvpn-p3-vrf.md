<!-- 
---
title: "dmvpn-p3-vrf"
author: "Gergő Téringer"
---
 -->
# Front Door Virtual Routing and Forwarding (FVRF)

```markdown
Virtual routing and forwarding (VRF) contexts create unique logical routers on a physical router so that router interfaces, routing tables, and forwarding tables are completely isolated from other VRF instances. In a DMVPN deployment, this means the routing table of one transport network (e.g., the Internet) is isolated from the routing table of another transport network (e.g., MPLS), and both are separated from the internal LAN routing table.

DMVPN tunnels are VRF-aware, meaning the tunnel's source and destination IP addresses (the underlay) can belong to a different VRF instance than the tunnel interface itself (the overlay). The VRF instance associated with the transport network is known as a Front Door VRF (FVRF). Using an FVRF instance for every DMVPN tunnel prevents route recursion because the transport and overlay networks remain in separate routing tables, ensuring encrypted packets always use the correct outbound physical interface.

> [!IMPORTANT]
> Modern Context & Compatibility: In modern enterprise edge designs—specifically on Cisco IOS-XE platforms like the Catalyst 8000 series or ASR 1000s—utilizing FVRF is considered a mandatory security best practice. It strictly segments the untrusted transport networks from the corporate overlay. This exact architecture is also the foundational building block for Cisco SD-WAN (Viptela), where the transport network is isolated in VPN 0 (the SD-WAN equivalent of an FVRF).

> [!TIP]
> VRF instances are locally significant to the router they are configured on. However, your configuration and naming conventions should remain strictly consistent across the entire enterprise to simplify operational aspects, automation, and troubleshooting.
```

## 1. Configuring FVRF and Interface Assignment

The following configurations demonstrate how to create an FVRF instance, initialize its address family, and assign it to a physical transport interface.

> [!WARNING]
> When a VRF instance is linked to an interface using the `vrf forwarding` command, any IP address currently configured on that interface is immediately removed by the IOS operating system. You must reconfigure the IP address after assigning the VRF.

```cisco
vrf definition INET01
 address-family ipv4
 exit
vrf definition INET02
 address-family ipv4
 exit
 
interface GigabitEthernet0/1
 vrf forwarding INET01
 ip address 172.16.31.1 255.255.255.252

interface GigabitEthernet0/2
 vrf forwarding INET02
 ip address dhcp
```

**Command Breakdown & Explanation:**

- `vrf definition`: Creates the VRF routing context (e.g., `INET01` and `INET02`).
- `address-family ipv4`: Initializes the IPv4 routing table within the defined VRF. (IPv6 can also be initialized here if required).
- `vrf forwarding`: Binds the physical interface to the specific FVRF instance. Traffic entering or leaving this interface is now subject exclusively to the VRF's routing table.
- `ip address dhcp`: Configures the interface to dynamically acquire its IP configuration (useful for broadband FVRF connections).

## 2. Associating the FVRF with the DMVPN Tunnel

Once the physical interface is placed into the FVRF, the DMVPN tunnel must be instructed to use that specific VRF to route its encrypted transport packets (the underlay).

> [!NOTE]
> A DMVPN tunnel can be associated with an FVRF (underlay) while simultaneously being part of an Inside VRF (IVRF/overlay). Both `vrf forwarding vrf-name` (overlay) and `tunnel vrf vrf-name` (underlay) can be used on the same tunnel interface. You must use different VRF names for this to be effective.

```cisco
interface tunnel 100
 tunnel vrf INET01

interface tunnel 200
 tunnel vrf INET02
```

**Command Breakdown & Explanation:**

- `tunnel vrf`: Binds the tunnel's transport (the encapsulated IPSec/GRE packets) to the FVRF instance. This tells the router to look in the FVRF routing table to resolve the path to the tunnel destination, effectively separating the underlay routing from the overlay routing.

## 3. FVRF Static Routing

Because the FVRF creates an isolated routing table, it requires its own default route to reach the Internet or service provider. FVRF interfaces assigned an IP address by DHCP automatically install a default route in the VRF table with an Administrative Distance (AD) of 254. FVRF interfaces with static IP addressing require a manual static default route configured explicitly for that VRF context.

```cisco
ip route vrf INET01 0.0.0.0 0.0.0.0 172.16.31.2
```

**Command Breakdown & Explanation:**

- `ip route vrf INET01`: Instructs the router to place this static route specifically into the `INET01` routing table rather than the global routing table.
- `0.0.0.0 0.0.0.0 172.16.31.2`: Creates a gateway of last resort pointing to the next-hop IP of the service provider within that transport network.

## 4. Troubleshooting and Verification

### 4.1 Verify VRF Initialization and Interface Binding

Validates that the VRF instances have been successfully created and that the correct physical interfaces have been assigned to them.

**Command:** `show vrf`

**Command Breakdown & Explanation:**
Checks the router's VRF database to ensure the logical isolation contexts exist and lists all interfaces bound to them.

What it checks and variables to look for:

- **Name**: Must be the configured FVRF name (e.g., `INET01` or `INET02`).
- **Protocols**: Must include `ipv4` (and `ipv6` if configured).
- **Interfaces**: Must list the correct physical transport interface (e.g., `Gi0/1`).

### 4.2 Verify FVRF Routing Table

Displays the isolated routing table for the specific Front Door VRF to ensure the transport network has reachability to the next-hop gateway.

**Command:** `show ip route vrf INET01`

**Command Breakdown & Explanation:**
Validates the presence of the static or DHCP-learned default route within the VRF context. Without a default route in the FVRF, the DMVPN tunnel cannot build its IPSec associations.

What it checks and variables to look for:

- **Gateway of last resort**: Must be `set to [next-hop-ip] to network 0.0.0.0`.
- **Code**: Must include `S*` (for static routes) or `Dh*` (if learned via DHCP on an interface like Gi0/2).
- **Interface**: The outgoing interface for the default route must match the FVRF-bound interface (e.g., `GigabitEthernet0/1`).
<!-- Created by: Gergő Téringer, 2026 -->