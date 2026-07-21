<!-- 
---
title: "SLAs and Tracking for FHRP"
author: "Gergő Téringer"
---
 -->
# SLAs and Tracking for FHRP

First-Hop Redundancy Protocols (FHRPs) natively monitor the state of the local interface they are configured on. However, if an upstream WAN link fails while the local LAN interface remains up, the router will continue acting as the active gateway, dropping all client traffic into a black hole.

To prevent this, we combine `IP SLA` to actively ping an upstream destination and `Object Tracking` to monitor that ping. If the ping fails, the tracking object goes down, which dynamically lowers the router's FHRP metrics, forcing a safe failover to the redundant router.

## 1. IP SLA Configuration

The first step is to generate synthetic traffic (an ICMP ping) to test upstream reachability. It is best practice to ping a reliable upstream IP, such as your ISP's next-hop gateway.

**Configuration:**

```cisco
ip sla 1
 icmp-echo 203.0.113.1 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now
```

**Command Breakdown & Explanation:**

- `ip sla 1`: Creates an IP SLA operation with the ID of `1`.
- `icmp-echo 203.0.113.1 ...`: Instructs the router to ping the upstream IP `203.0.113.1`. Specifying the `source-interface` ensures the ping is strictly routed out of the specific WAN link you want to test.
- `frequency 5`: Sets the router to send this ping every `5` seconds.
- `ip sla schedule 1 ...`: IP SLAs do not run until they are scheduled. This command tells operation `1` to start immediately and run continuously.

## 2. Object Tracking Configuration

IP SLA alone only generates statistics; it cannot trigger an action. You must create a tracking object that monitors the state of the IP SLA operation.

**Configuration:**

```cisco
track 10 ip sla 1 reachability
 delay down 10 up 20
```

**Command Breakdown & Explanation:**

- `track 10 ip sla 1 reachability`: Creates tracking object `10`. It is tied directly to the success or failure (reachability) of `ip sla 1`. If the ping succeeds, track `10` is UP. If it fails, track `10` is DOWN.
- `delay down 10 up 20`: A critical stability feature. It prevents the track from flapping during transient network blips. The router will wait `10` seconds after a ping fails before declaring the track DOWN, and it will wait `20` seconds after pings succeed again before declaring the track UP.

## 3. HSRP Tracking Example

Hot Standby Router Protocol (HSRP) uses a priority system to elect the Active router. By tying the tracking object to HSRP, we can automatically decrement the priority when the track goes down.

**Configuration:**

```cisco
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 track 10 decrement 20
```

**Command Breakdown & Explanation:**

- `standby 10 priority 110`: Sets this router's priority to `110`, making it the Active router (since the default is `100`).
- `standby 10 preempt`: Essential for tracking. It allows the standby router to immediately take over if its priority becomes higher than the current Active router.
- `standby 10 track 10 decrement 20`: Links tracking object `10` to HSRP group `10`. If the track goes DOWN, HSRP automatically subtracts `20` from the priority (`110 - 20 = 90`). Because `90` is lower than the standby router's default of `100`, the standby router immediately takes over forwarding duties.

## 4. GLBP Tracking Example

Gateway Load Balancing Protocol (GLBP) operates differently. While you can track the Active Virtual Gateway (AVG) priority, it is much more common and useful to track the Active Virtual Forwarder (AVF) weighting. This removes a router from load-balancing duty if its upstream link fails, without completely tearing down the AVG election.

**Configuration:**

```cisco
interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 glbp 20 ip 10.10.20.1
 glbp 20 priority 110
 glbp 20 preempt
 glbp 20 weighting 100 lower 85 upper 100
 glbp 20 weighting track 10 decrement 20
```

**Command Breakdown & Explanation:**

- `glbp 20 weighting 100 lower 85 upper 100`: Sets the maximum forwarding weight to `100`. The `lower 85` threshold means that if the weight drops to `85` or below, this router stops acting as an AVF and stops accepting client traffic. The `upper 100` threshold means it will resume forwarding only when its weight climbs back up to `100`.
- `glbp 20 weighting track 10 decrement 20`: Ties tracking object `10` to the GLBP weighting. If the tracked WAN link fails, the weight drops from `100` to `80`. Because `80` is below the configured lower threshold of `85`, GLBP safely redirects this router's share of the client traffic to the other redundant routers in the group.

## 5. Troubleshooting

Verifying IP SLA and Object Tracking integration with FHRP requires a step-by-step approach: first confirm the synthetic pings are successful, then verify the tracking object reflects that state, and finally ensure the FHRP group is applying the expected decrements.

### 5.1 Verifying IP SLA Statistics

This command confirms if the synthetic ICMP pings are actually being sent and whether they are receiving replies from the upstream gateway.

**Command:** `show ip sla statistics`

What it checks and variables to look for:

- `Number of successes` / `Number of failures`: A high failure count indicates the upstream IP is unreachable.
- `Latest RTT`: Displays the Round Trip Time of the last ping in milliseconds. If this shows `NoConnection` or `Timeout`, the SLA is failing.
- `Next operation start`: Confirms the schedule is active and the timer is ticking down to the next ping.

### 5.2 Verifying Object Tracking State

Once the SLA is confirmed, you must check the tracking object to ensure it is accurately reflecting the SLA's reachability and applying any configured delay timers.

**Command:** `show track`

What it checks and variables to look for:

- `Track 10`: The ID of the tracking object.
- `State`: Look for `Up` or `Down`. If it is `Down` but the SLA is successful, check your `delay` timers.
- `Change delayed`: If a state change is currently pending (e.g., waiting 10 seconds before officially declaring the track down), it will display the remaining countdown here.
- `Tracked by`: Lists the protocols (like `HSRP` or `GLBP`) that are actively monitoring this tracking object.

### 5.3 Verifying FHRP Decrement Action

Finally, confirm that the tracking object is actively influencing the FHRP priority or weighting as configured.

**Command:** `show standby/show glbp`

What it checks and variables to look for (HSRP):

- `Priority`: Shows the current active priority.
- `Track object 10 state`: Will display `Up` or `Down`. If it is `Down`, you will see `decrement 20`, and the active priority will instantly reflect this math.

What it checks and variables to look for (GLBP):

- `Weighting`: Displays the current weight, the configured maximum, and the upper/lower thresholds.
- `Track object 10 state`: Shows `Up` or `Down` and the configured decrement value. If `Down`, verify that the current weighting has fallen below the `lower` threshold, which correctly revokes the router's AVF status.

<!-- Created by: Gergő Téringer, 2026 -->