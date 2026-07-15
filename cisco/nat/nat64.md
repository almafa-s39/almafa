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
