<!-- 
---
title: "ISC-DHCP-SERVER (DHCPv6)"
author: "Gergő Téringer"
---
 -->
# ISC-DHCP-SERVER (DHCPv6)

> [!NOTE]
> This package is deprecated, so it is recommended to use
> `kea-dhcp6-server`, since that is the package actively being developed.
> (If you don't need advanced features, `isc-dhcp-server` still works
> well for DHCPv6.)
> This configuration will include DDNS as well, but you can leave those
> lines out if you want to run it without DDNS.

## 1. Install packages

```bash
apt install isc-dhcp-server
```

The `isc-dhcp-server` package ships both the DHCPv4 daemon (`dhcpd`) and
the DHCPv6 daemon (invoked as `dhcpd -6`), along with two separate
systemd services: `isc-dhcp-server` (IPv4) and `isc-dhcp-server6`
(IPv6). No separate package is required for DHCPv6.

## 2. Create listen

Edit `/etc/default/isc-dhcp-server` and set the `INTERFACESv6` variable
to the interface(s) the DHCPv6 daemon should listen on. This is the same
file used for the IPv4 daemon; IPv4 and IPv6 each have their own
variable (`INTERFACESv4` / `INTERFACESv6`) so the two protocols can be
bound to different interfaces if needed.

```bash
INTERFACESv6="eth0"
```

**Command Breakdown & Explanation:**

- `INTERFACESv6="eth0"`: Tells the `isc-dhcp-server6` service which
  interface(s) to bind to. Leave empty (`INTERFACESv6=""`) to disable the
  IPv6 daemon.

> [!NOTE]
> The interface must already have an IPv6 address configured from the
> subnet(s) you intend to serve; unlike DHCPv4, `dhcpd -6` will not add a
> usable address to the interface itself.

## 3. Create DDNS key

Create a TSIG key for updating your zone (the `tsig-keygen` utility is
included in the `bind9` package). This step is identical for DHCPv4 and
DHCPv6 — the key itself is protocol-agnostic and only secures
communication with the DNS server.

```bash
tsig-keygen "ddns" > /etc/dhcp/ddns.key
```

## 4. Configure the service

Edit `/etc/dhcp/dhcpd6.conf` (note the separate `dhcpd6.conf` file used
for DHCPv6, distinct from `dhcpd.conf`) and set the following lines. Key
differences from the IPv4 version: subnets use `subnet6` with a CIDR
prefix instead of `subnet ... mask ...`; there is no `option routers`
statement, since IPv6 routers are advertised via Router Advertisements
(RA), not DHCPv6; and the reverse DNS zone uses the `ip6.arpa` domain in
nibble format instead of `in-addr.arpa`.

```bash
include "/etc/dhcp/ddns.key";
ddns-update-style interim;

subnet6 2001:db8:10:10::/64 {
    range6 2001:db8:10:10::100 2001:db8:10:10::199;
    option dhcp6.name-servers 2001:db8:10:20::10, 2001:db8:10:20::11;
    zone unitel.com. {
        primary6 2001:db8:10:20::10;
        key "ddns";
    }
    zone 0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0.1.0.0.8.b.d.0.1.0.0.2.ip6.arpa. {
        primary6 2001:db8:10:20::10;
        key "ddns";
    }
    ddns-domainname "unitel.com.";
    ddns-rev-domainname "ip6.arpa.";
}
```

**Command Breakdown & Explanation:**

- `ddns-update-style interim;`: `interim` is the only DDNS update style
  ISC DHCP still supports (`standard`/`ad-hoc` were removed in current
  releases); this line is unchanged in meaning from the IPv4 version but
  the value must be `interim` for the daemon to start on a modern
  release such as Debian 13.
- `subnet6 2001:db8:10:10::/64`: Declares the IPv6 subnet to serve,
  using CIDR notation. There is no separate netmask keyword as there is
  in DHCPv4.
- `range6 2001:db8:10:10::100 2001:db8:10:10::199;`: Defines the pool of
  addresses handed out to clients within the subnet.
- `option dhcp6.name-servers ...;`: Provides DNS resolver addresses to
  clients. This replaces `option domain-name-servers` (the DHCPv4
  option); note there is no equivalent `option routers` for DHCPv6, as
  default gateway information comes from IPv6 Router Advertisements.
- `zone unitel.com. { primary6 2001:db8:10:20::10; key "ddns"; }`:
  Forward zone for DDNS updates. `primary6` (rather than `primary`)
  indicates the DNS master server is reachable at an IPv6 address.
- `zone 0.0.0...ip6.arpa. { ... }`: Reverse zone for `PTR` record
  updates, expressed as a nibble-reversed `ip6.arpa.` name rather than
  `in-addr.arpa.`. The example above corresponds to the
  `2001:db8:10:10::/64` prefix.
- `ddns-domainname "unitel.com.";`: Default forward-zone suffix appended
  to client hostnames.
- `ddns-rev-domainname "ip6.arpa.";`: Tells the server to construct
  reverse-zone updates using the `ip6.arpa.` suffix instead of the
  DHCPv4 default of `in-addr.arpa.`.

> [!TIP]
> IPv6 nibble-format reverse zones are tedious and error-prone to build
> by hand. Use a nibble/reverse-zone calculator, or generate the zone
> name with `dig -x <address>` against a resolver that already has the
> PTR delegation, to avoid transposition mistakes.
> [!IMPORTANT]
> `primary6` (rather than `primary`) is only required when the DNS
> master address itself is an IPv6 address, as shown above. If your
> authoritative DNS server is reachable only over IPv4 (e.g.
> `primary 10.10.20.10;`), keep using `primary`, even though the DHCP
> leases being updated are IPv6 — the master server's address family and
> the leased address family are independent of each other.

## 5. Failover peering

Unlike DHCPv4, ISC DHCP has never implemented a failover protocol for
DHCPv6. The IETF's `DHCPv6 Failover Protocol` (RFC 8156) exists as a
standard, but ISC DHCP only implements the earlier, expired DHCPv4
failover draft — the `failover peer` statement is rejected (or produces
undefined/unsupported behavior) when referenced from a `subnet6` or
`pool6` block. There is no `address6` / `peer address6` equivalent to
work around this.

Practical alternatives for DHCPv6 redundancy with `isc-dhcp-server`:

- Split the address pool between two independent, non-communicating
  servers (each configured with a disjoint `range6`), sized generously
  enough that a temporary outage of one server does not exhaust the
  other's pool. This trades efficiency for simplicity and avoids any
  need for state synchronization.
- Migrate to `kea-dhcp6-server`, which implements a modern High
  Availability hook (`libdhcp_ha.so`) that fully supports DHCPv6,
  including `hot-standby` and load-balancing modes. See the companion
  **"Kea DHCP6 Server"** document in this series for the equivalent HA
  configuration.

## 6. Verification and Troubleshooting

### 6.1 Verify DHCPv6 configuration syntax

```bash
dhcpd -6 -t -cf /etc/dhcp/dhcpd6.conf
```

What it checks and variables to look for:

- **Syntax output**: Should complete without printing
  `Configuration file errors encountered -- exiting`. Any JSON/parser
  error will report the offending line number in `dhcpd6.conf`.

### 6.2 Verify the DHCPv6 service status

```bash
systemctl status isc-dhcp-server6
```

What it checks and variables to look for:

- **Active**: Must be `active (running)`.
- **Loaded**: Must be `loaded (/lib/systemd/system/isc-dhcp-server6.service; enabled)`.

### 6.3 Verify network listening state on the DHCPv6 port

```bash
ss -tulnp | grep 547
```

What it checks and variables to look for:

- **State**: Must be `UNCONN` bound to `*:547` or `:::547`. DHCPv6
  clients send from UDP `546`; the server listens on UDP `547`. If
  nothing is bound to `547`, check `INTERFACESv6` in
  `/etc/default/isc-dhcp-server` and confirm `isc-dhcp-server6` (not
  only `isc-dhcp-server`) is enabled and running.
``

### 6.4 Verify DDNS updates are being sent

```bash
journalctl -u isc-dhcp-server6 -f
```

What it checks and variables to look for:

- **Added/updated DNS records**: Look for lines referencing
  `add_forward` and `add_reverse` (or corresponding `DHCID`/`fwd_state`
  messages) without adjacent `unable to add` errors. Frequent
  `unable to add` messages usually indicate the TSIG key name/secret in
  `ddns.key` does not match what is configured on the DNS server, or
  that the `zone` statements' `primary` / `primary6` address is
  unreachable.

<!-- Created by: Gergő Téringer, 2026 -->