<!-- 
---
title: "NAT64 & NAT46"
author: "Gergő Téringer"
---
-->
# NAT64 & NAT46

Network Address Translation 64 (NAT64) and NAT46 are transition mechanisms that allow communication between IPv6-only and IPv4-only networks. Stateful NAT64 is typically used when IPv6 clients need to initiate connections to IPv4 servers, while NAT46 allows legacy IPv4 clients to reach IPv6 resources.

## 1. Stateful NAT64 (IPv6 to IPv4)

The provided configuration establishes a stateful NAT64 translation. It allows an entire VLAN of IPv6 clients to access the IPv4 internet using a pool of IPv4 addresses, and also creates a dedicated static mapping for a specific host.

**Configuration:**

```cisco
nat64 prefix stateful 64:FF9C::/96

nat64 v4 pool POOL_NAT64 192.0.2.64 192.0.2.79

ipv6 access-list ACL_NAT64_VLAN201
 permit ipv6 2001:DB8:20:201::/64 any

nat64 v6v4 list ACL_NAT64_VLAN201 pool POOL_NAT64 overload

nat64 v6v4 static 2001:DB8:20:201::200 192.0.2.46

interface range g0/0-1,g0/3
 nat64 enable
```

**Command Breakdown & Explanation:**

- `nat64 prefix stateful 64:FF9C::/96`: Defines the Well-Known Prefix (WKP) used by NAT64. When the IPv6 client wants to reach an IPv4 address (like `192.0.2.1`), it sends traffic to `64:FF9C::192.0.2.1`. The router intercepts this prefix and translates it.
- `nat64 v4 pool POOL_NAT64 ...`: Creates a pool of available public IPv4 addresses (`192.0.2.64` through `192.0.2.79`) that the router will use to represent the internal IPv6 clients to the outside IPv4 world.
- `ipv6 access-list ACL_NAT64_VLAN201`: Defines which internal IPv6 networks are allowed to be translated. Here, it permits the entire `/64` subnet of VLAN 201.
- `nat64 v6v4 list ... overload`: Binds the ACL to the IPv4 pool and enables Port Address Translation (PAT) with the `overload` keyword. This allows many IPv6 clients to share the limited IPv4 addresses in the pool by using different source ports.
- `nat64 v6v4 static ...`: Creates a one-to-one static translation. The specific IPv6 host `2001:DB8:20:201::200` will always be translated to the IPv4 address `192.0.2.46`, which is useful if external IPv4 devices need to initiate a connection inbound to this specific IPv6 server.
- `interface range ...` and `nat64 enable`: Unlike legacy NAT which uses `ip nat inside` and `ip nat outside`, NAT64 simply requires enabling the feature on all participating interfaces, regardless of whether they face the IPv4 or the IPv6 network.

## 2. NAT46 (IPv4 to IPv6)

NAT46 (configured in Cisco as a `v4v6` static mapping) is used when a legacy IPv4-only host needs to initiate a connection to an IPv6-only server. By providing a phantom IPv4 address for the IPv4 client to target, the router can intercept and translate the traffic into IPv6.

**Configuration:**

```cisco
nat64 v4v6 static 192.0.2.100 2001:DB8:30:301::100

interface range g0/0-1
 nat64 enable
```

**Command Breakdown & Explanation:**

- `nat64 v4v6 static 192.0.2.100 2001:DB8:30:301::100`: Creates a static mapping that represents the IPv6 server (`2001:DB8:30:301::100`) as an IPv4 address (`192.0.2.100`) to the legacy network.
- `nat64 enable`: Just like the `v6v4` translation, the NAT64 process must be enabled on all transit interfaces handling this traffic.

**Practical Example:**
If an old IPv4-only printer or legacy application needs to send logs to a modern, IPv6-only Syslog server, you configure a NAT46 mapping. The legacy device sends its traffic to the destination IP `192.0.2.100`. The router intercepts this packet, translates the destination IP header to `2001:DB8:30:301::100`, translates the source IPv4 address using the configured NAT64 stateful prefix, and forwards it onto the IPv6 network.

## 3. Troubleshooting

Verifying NAT64 and NAT46 involves checking the active translation database, monitoring the prefix mappings, and confirming that the correct interfaces are participating in the translation process.

### 3.1 Verifying Active NAT64 Translations

This command displays the active stateful NAT64 translation table. It is crucial for confirming that IPv6 hosts are successfully mapping to IPv4 addresses and vice versa.

**Command:** `show nat64 translations`

What it checks and variables to look for:

- `Proto`: The transport protocol being translated (e.g., `TCP`, `UDP`, `ICMPv6`).
- `Original IPv4` / `Translated IPv4`: The public IPv4 address assigned from your NAT64 pool (or static mapping) and the actual destination IPv4 server address.
- `Original IPv6` / `Translated IPv6`: Your internal IPv6 client's global address and the synthesized IPv6 destination address (which includes the `64:FF9C::/96` prefix).

### 3.2 Verifying NAT64 Statistics

This command provides a high-level operational overview of the NAT64 engine, highlighting performance metrics and potential capacity issues.

**Command:** `show nat64 statistics`

What it checks and variables to look for:

- `Translations`: Displays the current number of active `Static` and `Dynamic` translations.
- `Packets translated`: A rising counter here confirms that the router is actively intercepting and converting IPv6/IPv4 headers.
- `Packets dropped`: High drop counters indicate issues. Drops could be caused by pool exhaustion (no IPv4 addresses left), ACL denials, or MTU/fragmentation problems during header translation.

### 3.3 Verifying NAT64 Prefixes and Interfaces

If translations are not occurring, it is often because the router is not properly synthesizing the addresses or listening on the correct interfaces.

**Command:**

```cisco
show nat64 prefix stateful
show nat64 interfaces
```

What it checks and variables to look for:

- `show nat64 prefix stateful`: Confirms that the router has successfully instantiated the stateful prefix (e.g., the Well-Known Prefix `64:FF9C::/96`) and is using it to route synthesized IPv6 traffic to the IPv4 network.
- `show nat64 interfaces`: Lists every physical or logical interface where the `nat64 enable` command has been applied. If an interface connecting to either the IPv4 or IPv6 domain is missing from this list, traffic will not be translated.

<!-- Created by: Gergő Téringer, 2026 -->