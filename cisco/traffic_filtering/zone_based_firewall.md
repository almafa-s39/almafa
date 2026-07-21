<!-- 
---
title: "Cisco Zone-Based Policy Firewall (ZBPF) Reference Guide"
author: "Gergő Téringer"
---
 -->
# Cisco Zone-Based Policy Firewall (ZBPF) Reference Guide

This document provides a comprehensive overview of the Cisco Zone-Based Policy Firewall (often referred to as ZBFW or ZBPF). It covers the core architecture, operational logic, default behaviors, and standard configuration methodology.

---

## 1. Introduction to Zone-Based Firewalls

Legacy Cisco firewalls (like CBAC) relied on interface-bound Access Control Lists (ACLs) and stateful inspection rules applied directly to physical or logical interfaces. As networks grew complex, managing bidirectional ACLs on multiple interfaces became cumbersome and error-prone.

**Zone-Based Policy Firewall (ZBPF)** changes this paradigm by abstracting security policies away from individual interfaces. Instead:

1. Interfaces are assigned to logical **Zones**.
2. Security policies are applied to **Zone Pairs** (the unidirectional flow of traffic between two zones).

This approach provides a highly scalable, easy-to-read, and default-deny security posture.

---

## 2. Core Components (C3PL)

ZBPF uses the Cisco Common Classification Policy Language (C3PL) to define traffic and actions. It consists of three hierarchical tiers:

### A. Class-Map (The "What")

A class-map identifies and categorizes the traffic you want to apply a policy to. Traffic can be matched using:

- Access Control Lists (ACLs)
- Protocols (TCP, UDP, ICMP, HTTP, etc.)
- Other class-maps (nested)

### B. Policy-Map (The "Action")

A policy-map dictates what the router should do with the traffic identified by the class-map. There are three primary actions:

- **Inspect:** Permits the traffic and creates a dynamic state table entry. Return traffic is automatically allowed back through the firewall. (Stateful inspection).
- **Pass:** Permits the traffic statelessly in one direction. It does *not* create a state table entry. Return traffic must be explicitly permitted by a reverse policy.
- **Drop:** Silently discards the traffic. (Note: `drop log` can be used to generate a syslog message when traffic is dropped).

### C. Zone-Pair (The "Where")

A zone-pair defines the unidirectional flow between a source zone and a destination zone (e.g., `INSIDE` to `OUTSIDE`). The Policy-Map is attached to the Zone-Pair.

---

## 3. Default Traffic Behaviors

Understanding the default routing behaviors of ZBPF is critical for troubleshooting:

| Traffic Flow | Default Behavior | Explanation |
| :--- | :--- | :--- |
| **Intra-Zone** (Same Zone to Same Zone) | **Permit** | Interfaces in the same zone can communicate freely without a policy. |
| **Inter-Zone** (Zone A to Zone B) | **Drop** | Traffic between different zones is implicitly denied unless a zone-pair policy explicitly allows it. |
| **No Zone to Zoned** | **Drop** | An interface *not* assigned to a zone cannot communicate with an interface assigned to a zone. |
| **No Zone to No Zone** | **Permit** | Interfaces without zone assignments route traffic normally (traditional router behavior). |

---

## 4. The "Self" Zone

The **Self Zone** is a system-defined zone that represents the router's own control plane and management plane (i.e., traffic originating from or destined to the router's own IP addresses).

- **Default Behavior:** Traffic to and from the Self zone is **Permitted** by default.
- **Exception:** If you configure a zone-pair involving the Self zone (e.g., `OUTSIDE` to `Self`), the default behavior instantly changes to **Drop** for that specific flow, and you must explicitly permit required management/routing traffic (like SSH, BGP, or OSPF).

---

## 5. Configuration Methodology (Step-by-Step)

The configuration of a ZBPF always follows a specific, logical order.

### Step 1: Create the Zones

Define the logical security boundaries.

```cisco
Router(config)# zone security INSIDE
Router(config)# zone security OUTSIDE
```

### Step 2: Define Traffic Classes (Class-Maps)

Identify the traffic to be inspected. Note the use of `type inspect`.

```cisco
! Use an ACL for highly specific traffic
Router(config)# ip access-list extended ACL_WEB_TRAFFIC
Router(config-ext-nacl)# permit tcp 10.0.0.0 0.255.255.255 any eq 80
Router(config-ext-nacl)# permit tcp 10.0.0.0 0.255.255.255 any eq 443

! Create a class-map to match the ACL
Router(config)# class-map type inspect match-all CM_WEB
Router(config-cmap)# match access-group name ACL_WEB_TRAFFIC

! Or, match protocols broadly (match-any means OR logic)
Router(config)# class-map type inspect match-any CM_GENERAL
Router(config-cmap)# match protocol tcp
Router(config-cmap)# match protocol udp
Router(config-cmap)# match protocol icmp
```

### Step 3: Define Policies (Policy-Maps)

Specify what happens to the traffic classes.

```cisco
Router(config)# policy-map type inspect PM_INSIDE_TO_OUTSIDE
Router(config-pmap)# class type inspect CM_WEB
Router(config-pmap-c)# inspect
Router(config-pmap)# class type inspect CM_GENERAL
Router(config-pmap-c)# inspect
Router(config-pmap)# class class-default
Router(config-pmap-c)# drop log
```

>[!NOTE]
> `class-default` drops traffic by default, but adding `drop log` is highly recommended for visibility.

### Step 4: Create Zone-Pairs and Apply Policies

Bind the source zone, destination zone, and policy together.

```cisco
Router(config)# zone-pair security ZP_IN_TO_OUT source INSIDE destination OUTSIDE
Router(config-sec-zone-pair)# service-policy type inspect PM_INSIDE_TO_OUTSIDE
```

### Step 5: Assign Interfaces to Zones

Activate the firewall by placing interfaces into their respective zones.

```cisco
Router(config)# interface GigabitEthernet0/1
Router(config-if)# zone-member security INSIDE

Router(config)# interface GigabitEthernet0/2
Router(config-if)# zone-member security OUTSIDE
```

---

## 6. Troubleshooting

Verifying Zone-Based Policy Firewalls involves confirming that interfaces are correctly assigned to their respective zones, that zone-pairs are established, and checking the real-time hit counters to ensure traffic is being inspected or dropped appropriately.

### 6.1 Verifying Zones and Interface Assignments

This command provides a high-level overview of your defined security zones and which physical or logical interfaces belong to them.

**Command:** `show zone security`

What it checks and variables to look for:

- `zone`: Lists the name of the created zone (e.g., `INSIDE`, `OUTSIDE`, and the system `self` zone).
- `Member Interfaces`: Confirms which interfaces are actively bound to the zone. If an interface is missing here, it is currently operating under the "No Zone" default routing behavior and will drop traffic destined for a zoned interface.

### 6.2 Verifying Zone-Pairs

This command verifies the directional flow you have established between your zones and confirms which service policy is attached to that specific flow.

**Command:** `show zone-pair security`

What it checks and variables to look for:

- `Source-Zone` / `Destination-Zone`: Confirms the unidirectional path (e.g., source `INSIDE` destination `OUTSIDE`).
- `service-policy`: Shows the name of the policy-map applied to this pair. If this is empty, the zone-pair exists but is implicitly dropping all inter-zone traffic.

### 6.3 Verifying Policy Action and Active Sessions

This is the most critical command for ZBPF troubleshooting. It digs into the C3PL hierarchy, showing exactly how much traffic is hitting each class-map and whether it is being passed, dropped, or inspected.

**Command:** `show policy-map type inspect zone-pair`

What it checks and variables to look for:

- `Class-map`: Lists the traffic classes being evaluated (e.g., `CM_WEB`, `CM_GENERAL`).
- `Match`: Displays the ACL or protocol being matched.
- `Action`: Shows `Inspect`, `Pass`, or `Drop`.
- `Packet / Byte Counters`: Look for increments here. If traffic is failing but the counters for your permitted class-map are `0`, the traffic is either not hitting the firewall or the ACL matching logic is flawed.
- `Session creations`: Shows the active stateful connections currently being tracked by the firewall.
- `Class-map: class-default (match-any)`: Pay close attention to the drop counters here to identify legitimate traffic that is being silently blocked by the firewall's default deny posture.

<!-- Created by: Gergő Téringer, 2026 -->