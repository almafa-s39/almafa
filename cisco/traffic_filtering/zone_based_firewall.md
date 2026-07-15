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

## 6. Verification and Troubleshooting Commands

Use the following commands to monitor and troubleshoot the firewall state:

- **`show zone security`**: Displays all configured zones and the interfaces assigned to them.
- **`show zone-pair security`**: Displays all configured zone-pairs and the policies attached to them.
- **`show policy-map type inspect zone-pair`**: The most critical troubleshooting command. Displays hit counts, drop counts, and active inspection sessions for a specific zone-pair.
- **`show policy-firewall stats`**: Shows global statistics for the firewall engine.
