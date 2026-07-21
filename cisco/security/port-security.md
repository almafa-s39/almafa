<!-- 
---
title: "Port-securty & edge protection"
author: "Gergő Téringer"
---
-->
# Port-securty & edge protection

Port security and edge protection features are critical for securing the access layer of a network. They prevent unauthorized devices from connecting, mitigate MAC flooding attacks, and protect the Spanning Tree Protocol (STP) topology from rogue switches.

## 1. Cisco IOS Advanced Port Security Guide

Port security offers highly granular control over interface access. Beyond simply limiting the number of devices, administrators can dictate exactly how the switch learns MAC addresses, how it reacts to unauthorized connections, and how long it remembers authorized devices.

### 1.1 MAC Address Learning Methods

The switch can learn MAC addresses in three distinct ways: Dynamic (the default), Static, and Sticky. Mixing these methods provides flexibility for different network environments.

**Configuration:**

```cisco
interface g0/2
 switchport port-security
 switchport port-security maximum 3
 switchport port-security mac-address 0050.56XX.XXXX
 switchport port-security mac-address sticky
```

**Command Breakdown & Explanation:**

- `switchport port-security maximum 3`: Sets the total limit of allowed MAC addresses.
- `switchport port-security mac-address 0050.56XX.XXXX`: Defines a Static MAC address. This specific device is permanently allowed on this port and will not be lost if the switch reboots. This counts as one of the three maximum allowed addresses.
- `switchport port-security mac-address sticky`: Enables Sticky learning. The switch will dynamically learn the next connected MAC addresses (up to the maximum limit) and convert them into sticky secure MAC addresses. These are automatically written to the running configuration, allowing them to be permanently saved via `write memory`.

**Practical Example:**
If you have a critical server that must never be blocked, you configure its MAC statically. If you have two regular PCs attached to a phone on the same port, you enable `sticky` so the switch learns their MACs automatically, but locks them in so nobody else can plug into that port later.

### 1.2 Security Violation Modes

When a device violates the port security policy (e.g., exceeding the maximum MAC limit or a static MAC appearing on the wrong port), the switch takes action based on the configured violation mode. There are three modes available: Shutdown, Restrict, and Protect.

**Configuration:**

```cisco
interface g0/3
 switchport port-security
 switchport port-security violation restrict
```

**Command Breakdown & Explanation:**

- `violation shutdown`: This is the default mode. The interface is placed into an `err-disabled` state, shutting off all traffic. An SNMP trap is generated, and a syslog message is logged.
- `violation restrict`: The port remains up, and traffic from valid MAC addresses continues to forward normally. However, packets from unknown MAC addresses are dropped. The switch generates a syslog message, sends an SNMP trap, and increments the security violation counter.
- `violation protect`: The port remains up, and traffic from unauthorized MAC addresses is silently dropped. No syslog message is generated, no SNMP trap is sent, and the violation counter is not incremented.

**Practical Example:**
In a high-security environment, `shutdown` is preferred to completely isolate the threat. In a busy office environment where a complete port shutdown would disrupt a legitimate user sharing a port with an unauthorized device, `restrict` is ideal. It blocks the rogue device while alerting the administration team, without taking down the entire interface.

### 1.3 Port Security Aging

By default, dynamically learned secure MAC addresses remain on the port until the switch restarts or the MAC is manually cleared. Aging allows the switch to automatically remove old MAC addresses, freeing up space for new devices without manual intervention.

**Configuration:**

```cisco
interface g0/4
 switchport port-security
 switchport port-security maximum 5
 switchport port-security aging time 10
 switchport port-security aging type inactivity
```

**Command Breakdown & Explanation:**

- `switchport port-security aging time 10`: Sets the aging timer to `10` minutes.
- `switchport port-security aging type inactivity`: Dictates how the timer functions.
- `type absolute`: (The default if not specified) The MAC address is removed exactly `10` minutes after it is learned, regardless of whether the device is still transmitting data.
- `type inactivity`: The `10` minute timer resets every time the switch receives a frame from the MAC address. The MAC is only removed if the device is completely silent for the full 10 minutes.

**Practical Example:**
Aging is highly useful in a conference room or a hot-desk environment. Setting the type to `inactivity` ensures that as long as a user is working, their MAC address stays authorized. Once they disconnect and leave the room for 10 minutes, the switch drops their MAC, making room for the next person to plug in without hitting the maximum limit.

## 2. Error-Disable Auto-Recovery

When a violation occurs (like exceeding the MAC limit), the port shuts down and normally requires a network administrator to manually log in and bounce the port. Configuring auto-recovery reduces downtime for accidental violations.

**Configuration:**

```cisco
errdisable recovery interval 180
errdisable recovery cause psecure-violation
```

**Command Breakdown & Explanation:**

- `errdisable recovery interval 180`: Sets a global timer instructing the switch to wait `180` seconds (3 minutes) before attempting to automatically re-enable any port that is in an `err-disabled` state.
- `errdisable recovery cause psecure-violation`: Explicitly tells the switch that it is allowed to use the auto-recovery timer for ports that were shut down specifically due to a port security violation.

**Practical Example:**

If an employee brings an unauthorized mini-switch from home, connects it to `g0/2`, and plugs in three laptops, the switch will detect more than `2` MAC addresses. Port security will instantly trigger, placing the port into `err-disabled` and cutting off access. If the employee realizes their mistake and unplugs the mini-switch, they do not need to call the helpdesk; the port will automatically reset and turn itself back on after `180` seconds.

## 3. Troubleshooting

### 3.1 Verifying Interface Security Status

This command provides a detailed look at the port security settings and current operational status for a specific interface.

What it checks and variables to look for:

- `Port Security`: Confirms if the feature is `Enabled` or `Disabled`.
- `Port Status`: Shows if the port is `Secure-up`, `Secure-down`, or `Secure-shutdown` (which means it tripped a violation).
- `Violation Mode`: Confirms the configured reaction (`Shutdown`, `Restrict`, or `Protect`).
- `Maximum MAC Addresses`: The configured limit.
- `Total MAC Addresses`: How many addresses the switch has currently learned on this port.
- `Security Violation Count`: Increments if the mode is set to Restrict or Shutdown and unauthorized frames are detected.
**Command:** `show port-security interface g0/2`

### 3.2 Verifying the Secure MAC Address Table

To view exactly which MAC addresses have been learned and how they were learned, you must check the port security address table, not just the standard MAC address table.

**Command:** `show port-security address`

What it checks and variables to look for:

- `Vlan` and `Mac Address`: The exact hardware addresses authorized on the switch.
- `Type`: Indicates how the switch learned the address (e.g., `SecureDynamic`, `SecureStatic`, or `SecureSticky`).
- `Ports`: The interface to which this MAC address is securely bound.
- `Remaining Age (mins)`: If aging is configured, this shows the countdown until the MAC address is flushed from the secure table.

### 3.3 Checking Error-Disabled Ports

If a port went down and you need to see why, this command gives you a fast overview of all interfaces in the `err-disabled` state and the specific reason they were shut down.

**Command:** `show interfaces status err-disabled`

What it checks and variables to look for:

- `Port`: The specific interface that is down.
- `Status`: Will display `err-disabled`.
- `Reason`: If tripped by this feature, it will explicitly state `psecure-violation`. (This can also show other reasons like `bpduguard` if someone plugged in an unauthorized switch on an edge port).

<!-- Created by: Gergő Téringer, 2026 -->