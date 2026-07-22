<!-- 
---
title: "IPv4 NAT"
author: "Gergő Téringer"
---
 -->
<!-- 
---
title: "IPv4 NAT"
author: "Gergő Téringer"
---
-->
# IPv4 NAT

IPv4 Network Address Translation (NAT) is essential for preserving public IP space and providing internet access to internal networks utilizing private RFC 1918 addressing. The following configurations cover the most common enterprise implementations: interface overload, address pools, and static mappings.

## 1. Dynamic NAT Overload (PAT) to an Interface

Port Address Translation (PAT), commonly referred to as NAT overload, is the most frequently used translation method. It allows an entire internal network to share a single public IP address by mapping multiple private IP addresses to unique source ports on the outside interface.

**Configuration:**

```cisco
interface GigabitEthernet0/0
 description LAN-FACING
 ip nat inside

interface GigabitEthernet0/1
 description WAN-FACING
 ip nat outside

access-list 10 permit 10.10.0.0 0.0.255.255

ip nat inside source list 10 interface GigabitEthernet0/1 overload
```

**Command Breakdown & Explanation:**

- `ip nat inside` and `ip nat outside`: These commands establish the NAT boundary. The router must know which interface connects to the private network and which connects to the public internet.
- `access-list 10 permit 10.10.0.0 0.0.255.255`: Defines the interesting traffic. This standard ACL permits any internal host within the `10.10.0.0/16` subnet to be translated.
- `ip nat inside source list 10 interface GigabitEthernet0/1 overload`: Binds the ACL to the physical outside interface and explicitly enables PAT using the `overload` keyword.

**Practical Example:**
If fifty employees in the `10.10.0.0/16` subnet browse the web simultaneously, the router translates all their private IPs to the single public IP assigned to `GigabitEthernet0/1`. It tracks each individual session using different randomized port numbers.

## 2. Dynamic NAT with an IP Address Pool (Overload)

When a single public IP address is not enough to handle the sheer volume of outbound connections (due to port exhaustion), you can configure a NAT pool. This allows the router to distribute PAT sessions across multiple public IP addresses provided by your ISP.

**Configuration:**

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside

ip nat pool PUBLIC_POOL 198.51.100.10 198.51.100.20 netmask 255.255.255.0

access-list 20 permit 10.20.0.0 0.0.255.255

ip nat inside source list 20 pool PUBLIC_POOL overload
```

**Command Breakdown & Explanation:**

- `ip nat pool PUBLIC_POOL ...`: Creates a pool of available public IPv4 addresses named `PUBLIC_POOL`. It defines the starting IP (`198.51.100.10`), the ending IP (`198.51.100.20`), and the subnet mask.
- `access-list 20 permit ...`: Defines the internal subnet (`10.20.0.0/16`) authorized to use this pool.
- `ip nat inside source list 20 pool PUBLIC_POOL overload`: Connects the ACL to the NAT pool. Crucially, keeping the `overload` keyword ensures that the router will PAT (share ports) across all IPs in the pool. Without `overload`, it would be a 1:1 dynamic mapping and would fail after 11 users connected.

## 3. Static NAT (One-to-One)

Static NAT creates a fixed, permanent mapping between a specific internal private IP address and a specific external public IP address. This is required for servers that must be reachable from the outside world, as the translation never changes or times out.

**Configuration:**

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside

ip nat inside source static 10.10.10.5 198.51.100.5
```

**Command Breakdown & Explanation:**

- `ip nat inside source static 10.10.10.5 198.51.100.5`: Directly maps the internal server's private IP (`10.10.10.5`) to a dedicated public IP (`198.51.100.5`).

**Practical Example:**
If you host a web server on `10.10.10.5`, an external user can type `198.51.100.5` into their browser. The router receives the packet, translates the destination IP back to `10.10.10.5`, and routes it internally.

## 4. Static PAT (Port Forwarding)

If you only have one public IP address available but still need to host internal services, you can use Static PAT (Port Forwarding). This maps a specific TCP or UDP port on your public IP to a specific port on an internal private IP.

**Configuration:**

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside

ip nat inside source static tcp 10.10.10.25 80 interface GigabitEthernet0/1 80
ip nat inside source static tcp 10.10.10.30 22 interface GigabitEthernet0/1 2222
```

**Command Breakdown & Explanation:**

- `... static tcp 10.10.10.25 80 interface GigabitEthernet0/1 80`: Tells the router that if any external traffic hits the public IP of `GigabitEthernet0/1` on TCP port `80` (HTTP), it should immediately forward it to the internal server `10.10.10.25` on port `80`.
- `... static tcp 10.10.10.30 22 interface GigabitEthernet0/1 2222`: This is an example of port translation. If external traffic hits the public IP on TCP port `2222`, it is translated and forwarded to the internal management server `10.10.10.30` on standard SSH port `22`, adding a layer of obfuscation.

## 5. Troubleshooting

Verifying IPv4 NAT involves checking the active translation table and confirming that the router is correctly translating traffic based on the configured rules and access control lists.

### 5.1 Verifying Active NAT Translations

This command displays every active translation currently being processed by the router, which is essential for confirming that internal hosts are actually reaching the outside world.

**Command:** `show ip nat translations`

What it checks and variables to look for:

- `Pro`: The protocol being translated (e.g., `tcp`, `udp`, `icmp`).
- `Inside local`: The actual private IP address and port of your internal host before translation.
- `Inside global`: The public IP address and port that the router has translated the internal host into.
- `Outside local` / `Outside global`: The IP address of the external destination server. (These are usually identical unless you are performing complex dual-NAT scenarios).

### 5.2 Verifying NAT Statistics

This command provides a high-level overview of the NAT configuration, showing operational statistics and pool utilization.

**Command:** `show ip nat statistics`

What it checks and variables to look for:

- `Total active translations`: Shows the breakdown of `Static`, `Dynamic`, and `Extended` (PAT) translations currently active.
- `Hits` / `Misses`: A high number of hits means NAT is actively working. Misses indicate packets that needed translation but failed (often due to pool exhaustion).
- `Dynamic mapping`: Displays the configured ACLs and the interfaces or pools they are mapped to, verifying your configuration is active.
- `Pool stats`: If using a NAT pool, this shows the total addresses, the percentage of addresses currently allocated, and how many times the pool failed to allocate an IP.

### 5.3 Real-Time NAT Debugging

If translations do not appear in the table, you can monitor the NAT process in real-time to see exactly which packets are being intercepted and translated. (Warning: Use this cautiously on production routers with high traffic).

**Command:** `debug ip nat`

What it checks and variables to look for:

- `s=`: The source IP address. Watch how it changes from the private IP to the public IP as it routes from the inside interface to the outside interface.
- `d=`: The destination IP address.
- `*` (Asterisk): If you see an asterisk next to NAT output, it indicates the packet was processed in the fast path (hardware switching) rather than the process path (CPU).

<!-- Created by: Gergő Téringer, 2026 -->