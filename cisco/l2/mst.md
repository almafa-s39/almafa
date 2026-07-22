<!-- 
---
title: "Multiple Spanning Tree (MST) Configuration"
author: "Gergő Téringer"
---
 -->
# Multiple Spanning Tree (MST) Configuration

This document outlines the procedure to enable and configure the Multiple Spanning Tree (MST) protocol (IEEE 802.1s) on a Cisco switch. MST allows multiple VLANs to be mapped to a single spanning-tree instance, significantly reducing CPU and memory overhead compared to Per-VLAN Spanning Tree (PVST+).

> [!IMPORTANT]
> Modern Context & Compatibility: While modern Data Center architectures (like Cisco Nexus Spine-Leaf with VXLAN/EVPN or Cisco ACI) rely on Layer 3 routing to eliminate Layer 2 loops entirely, MST remains the gold standard for traditional Layer 2 campus LANs. It integrates safely with standard server OS teaming topologies (like Windows Server 2025 NIC Teaming in Switch Independent mode or standard LACP) and prevents broadcast storms that can cripple modern hypervisor environments.

## 1. Global MST Configuration

The first step is to change the spanning-tree mode to MST and map specific VLANs to an MST instance.

```cisco
spanning-tree mode mst
spanning-tree mst configuration
 instance 10 vlan 10
```

**Command Breakdown & Explanation:**

- `spanning-tree mode mst`: Changes the global spanning-tree operating mode from the default (usually PVST+ or Rapid-PVST+) to MST. This affects the entire switch and causes a temporary recalculation of the spanning-tree topology.
- `spanning-tree mst configuration`: Enters the MST configuration submode where the region name, revision number, and VLAN-to-instance mappings are defined.
- `instance 10 vlan 10`: Maps VLAN 10 to MST Instance 10 (MSTI 10). By default, all VLANs belong to Instance 0 (the Internal Spanning Tree, or IST).

> [!TIP]
> For MST to form a proper region between multiple switches, the `name`, `revision`, and VLAN-to-instance mappings must match exactly across all interconnected switches.

## 2. Root Bridge Configuration

To ensure deterministic traffic flow, the root bridge placement must be manually enforced for both the MST instances and any legacy PVST+ domains that might still be interacting with the switch via the IST (Instance 0).

```cisco
spanning-tree mst 10 root primary
spanning-tree vlan 1-4094 root primary
```

**Command Breakdown & Explanation:**

- `spanning-tree mst 10 root primary`: A macro command that automatically lowers the bridge priority for MST Instance 10 to a value (typically 24576 or lower) that guarantees the switch becomes the root bridge for that instance. Use the `secondary` keyword on your designated backup root switch.
- `spanning-tree vlan 1-4094 root primary`: A legacy PVST+/Rapid-PVST+ macro command. It lowers the bridge priority for all individual VLANs. In a pure MST environment, this command applies to the IST boundary, but it is primarily used during migrations or as a fallback safeguard to ensure this switch remains the root if the spanning-tree mode is accidentally reverted to PVST+.

## 3. Troubleshooting and Verification

### 3.1 Verify MST Instance Status

Displays the current spanning-tree topology, root bridge identity, and port states for a specific MST instance.

**Command:** `show spanning-tree mst 10`

**Command Breakdown & Explanation:**
Validates that the switch is operating in MST mode and checks whether it has successfully assumed the root bridge role for the specified instance.

What it checks and variables to look for:

- **Root**: Should indicate `This bridge is the root` if the primary command was successful.
- **Status**: Port roles must show `FWD` (Forwarding) for designated and root ports, or `BLK` / `ALT` for blocked loop-prevention ports.

### 3.2 Verify MST Configuration and Region

Displays the current MST region configuration and the exact VLAN-to-instance mappings.

**Command:** `show spanning-tree mst configuration`

**Command Breakdown & Explanation:**
Ensures that the VLANs are correctly mapped to their intended MST instances. This output must be identical across all switches intended to be in the same MST region.

What it checks and variables to look for:

- **Name**: Must be the configured region string (defaults to the switch MAC address if left unconfigured).
- **Revision**: Must match the configured revision number across the topology.
- **Instance**: Must show `10` mapped to VLAN `10`.

<!-- Created by: Gergő Téringer, 2026 -->