# Cisco IOS PPPoE Server and Client Configuration Guide

This document covers configuring a Cisco IOS router as both a PPPoE
server (acting as a Broadband Remote Access Server / Access
Concentrator) and a PPPoE client (acting as Customer Premises
Equipment connecting to an upstream provider). PPPoE (PPP over
Ethernet) encapsulates PPP frames inside Ethernet frames, allowing
per-subscriber authentication, IP addressing, and accounting over what
is otherwise a shared, connectionless Ethernet medium — commonly used
by ISPs over DSL/fiber access networks, and equally useful in lab or
enterprise environments to simulate that same model.

The commands in this guide apply to Cisco IOS and IOS XE software
trains that support `bba-group pppoe` and `pppoe-client` (IOS 12.4T and
later, IOS XE 3.x/16.x/17.x). Always confirm exact syntax against the
Cisco documentation for your specific platform and software release.

## 1. PPPoE Server Configuration

The server side is built from four pieces: an address pool for
clients, a Virtual-Template interface that defines the PPP session
parameters, a BBA (Broadband Access) group that ties PPPoE sessions to
that template, and the physical interface facing the clients.

### 1.1 Loopback and Client Address Pool

The Virtual-Template interface needs an IP address to negotiate from,
borrowed from a Loopback interface via `ip unnumbered`, and a pool of
addresses to hand out to connecting clients.

```cisco
interface Loopback0
 ip address 192.168.100.1 255.255.255.0
!
ip local pool PPPOE-POOL 192.168.100.10 192.168.100.100
```

**Command Breakdown & Explanation:**

- `interface Loopback0` / `ip address 192.168.100.1 255.255.255.0`:
  Creates a stable, always-up interface whose address the Virtual-
  Template borrows. Using a loopback (rather than assigning the address
  directly to the physical or Virtual-Template interface) keeps the
  server's own address stable regardless of session churn.
- `ip local pool PPPOE-POOL 192.168.100.10 192.168.100.100`: Defines a
  named pool of addresses, `PPPOE-POOL`, that will be handed out one
  per client session.

### 1.2 Virtual-Template Interface

The Virtual-Template defines the settings every cloned PPP session
inherits when a client connects: the IP addressing behavior and the
authentication method.

```cisco
interface Virtual-Template1
 ip unnumbered Loopback0
 peer default ip address pool PPPOE-POOL
 ppp authentication chap
 ppp mtu adaptive
```

**Command Breakdown & Explanation:**

- `interface Virtual-Template1`: Creates template `1`. Cisco IOS clones
  a new Virtual-Access interface from this template for each PPPoE
  session that comes up.
- `ip unnumbered Loopback0`: Borrows its IP address from `Loopback0`
  rather than owning a dedicated address itself.
- `peer default ip address pool PPPOE-POOL`: Assigns each connecting
  client an address from the `PPPOE-POOL` pool defined in Section 1.1.
- `ppp authentication chap`: Requires clients to authenticate via CHAP
  before a session is established. `pap` or `chap pap` (accept either)
  are also valid depending on client capability.
- `ppp mtu adaptive`: Allows the negotiated PPP MTU to adjust
  automatically to account for the PPPoE encapsulation overhead,
  reducing manual MTU troubleshooting.

> [!NOTE]
> Local usernames/passwords for CHAP authentication are configured with
> the global `username <name> password <secret>` command, matching the
> hostname each client sends in its CHAP response. For larger
> deployments, `aaa new-model` with a RADIUS server is used instead of
> local usernames — outside the scope of this guide.

### 1.3 BBA Group (PPPoE Profile)

The BBA group is the PPPoE-specific profile that links incoming PPPoE
discovery/session traffic to the Virtual-Template above.

```cisco
bba-group pppoe PPPOE-GROUP
 virtual-template 1
```

**Command Breakdown & Explanation:**

- `bba-group pppoe PPPOE-GROUP`: Creates a named PPPoE profile,
  `PPPOE-GROUP`. A `global` profile (created with `bba-group pppoe
  global`) is also available as a catch-all default if no named group
  is referenced on an interface.
- `virtual-template 1`: Associates this profile with `Virtual-
  Template1` from Section 1.2 — every session accepted under this BBA
  group clones from that template.

### 1.4 Enabling PPPoE on the Physical Interface

Finally, PPPoE is enabled on the physical (or subinterface) facing the
clients, and linked to the BBA group.

```cisco
interface GigabitEthernet0/1
 description WAN-facing interface toward PPPoE clients
 no ip address
 pppoe enable group PPPOE-GROUP
 no shutdown
```

**Command Breakdown & Explanation:**

- `no ip address`: The physical interface itself carries no IP address
  — addressing happens entirely at the PPP/Virtual-Access layer once a
  session comes up.
- `pppoe enable group PPPOE-GROUP`: Enables PPPoE session negotiation
  on this interface and binds it to the `PPPOE-GROUP` BBA group from
  Section 1.3.

### 1.5 Full Server Configuration Example

```cisco
interface Loopback0
 ip address 192.168.100.1 255.255.255.0
!
ip local pool PPPOE-POOL 192.168.100.10 192.168.100.100
!
username client1 password 0 CHANGE-ME-STRONG-PASSWORD
!
bba-group pppoe PPPOE-GROUP
 virtual-template 1
!
interface Virtual-Template1
 ip unnumbered Loopback0
 peer default ip address pool PPPOE-POOL
 ppp authentication chap
 ppp mtu adaptive
!
interface GigabitEthernet0/1
 description WAN-facing interface toward PPPoE clients
 no ip address
 pppoe enable group PPPOE-GROUP
 no shutdown
```

## 2. PPPoE Client Configuration

The client side has two parts: the physical interface that carries the
PPPoE discovery/session traffic toward the access concentrator, and a
logical Dialer interface where PPP negotiation, authentication, and IP
addressing actually happen.

### 2.1 Physical Interface (Dial Pool Association)

```cisco
interface GigabitEthernet0/0
 description WAN link to ISP / PPPoE Access Concentrator
 no ip address
 pppoe enable group global
 pppoe-client dial-pool-number 1
 no shutdown
```

**Command Breakdown & Explanation:**

- `no ip address`: As on the server side, the physical interface itself
  carries no IP address.
- `pppoe enable group global`: Enables PPPoE client discovery on this
  interface. On many IOS releases, plain `pppoe enable` (with no
  `group` keyword) is also accepted and behaves the same way — check
  `pppoe enable ?` on your platform if the `group` keyword is rejected.
- `pppoe-client dial-pool-number 1`: Links this physical interface to
  dialer pool `1`, matching the `dialer pool 1` statement configured on
  the Dialer interface in Section 2.2. This is what tells IOS which
  Dialer interface should be used once a PPPoE session forms here.

### 2.2 Dialer Interface (PPP Negotiation and Authentication)

```cisco
interface Dialer1
 mtu 1492
 ip address negotiated
 ip nat outside
 encapsulation ppp
 dialer pool 1
 dialer-group 1
 ppp authentication chap callin
 ppp chap hostname client1
 ppp chap password 0 CHANGE-ME-STRONG-PASSWORD
```

**Command Breakdown & Explanation:**

- `mtu 1492`: Reduces the interface MTU from the Ethernet default of
  `1500` to `1492`, accounting for the 8-byte PPPoE header.
- `ip address negotiated`: Obtains the interface's IP address from the
  PPP/IPCP negotiation with the access concentrator, rather than a
  static or DHCP address.
- `ip nat outside`: Marks this interface as the outside (public-facing)
  side for NAT overload, if this router is also performing NAT for an
  internal LAN. Omit if this router is not doing NAT.
- `encapsulation ppp`: Sets PPP as the Layer 2 protocol carried over
  this dialer interface.
- `dialer pool 1`: Matches the `pppoe-client dial-pool-number 1`
  statement from Section 2.1, binding this Dialer interface to that
  physical interface.
- `dialer-group 1`: References dialer access-list group `1`, defined
  separately with the `dialer-list` command (Section 2.3), which
  specifies what counts as "interesting traffic" for bringing the
  interface up.
- `ppp authentication chap callin`: Authenticates using CHAP.
  `callin` restricts authentication to only when this router is
  answering (dialing in), which is the correct behavior for a client
  that never accepts inbound sessions itself.
- `ppp chap hostname client1` / `ppp chap password 0
  CHANGE-ME-STRONG-PASSWORD`: The CHAP username/password credentials
  supplied to the access concentrator; must match an account configured
  there (Section 1.2's `username` command, in a lab scenario mirroring
  this guide's server side).

> [!TIP]
> If your provider uses PAP instead of CHAP, replace the three `ppp
> chap ...` lines with `ppp authentication pap callin` and `ppp pap
> sent-username client1 password 0 CHANGE-ME-STRONG-PASSWORD`.

### 2.3 Default Route via Dialer Interface

```cisco
dialer-list 1 protocol ip permit
!
ip route 0.0.0.0 0.0.0.0 Dialer1
```

**Command Breakdown & Explanation:**

- `dialer-list 1 protocol ip permit`: Defines dialer access-list `1`
  (referenced by `dialer-group 1` above) to treat all IP traffic as
  interesting, meaning any IP packet can trigger/keep the dialer
  interface active.
- `ip route 0.0.0.0 0.0.0.0 Dialer1`: Installs a default route pointing
  at the Dialer interface, so all traffic without a more specific route
  is sent out over the PPPoE session once it is established.

### 2.4 Full Client Configuration Example

```cisco
interface GigabitEthernet0/0
 description WAN link to ISP / PPPoE Access Concentrator
 no ip address
 pppoe enable group global
 pppoe-client dial-pool-number 1
 no shutdown
!
interface Dialer1
 mtu 1492
 ip address negotiated
 ip nat outside
 encapsulation ppp
 dialer pool 1
 dialer-group 1
 ppp authentication chap callin
 ppp chap hostname client1
 ppp chap password 0 CHANGE-ME-STRONG-PASSWORD
!
dialer-list 1 protocol ip permit
!
ip route 0.0.0.0 0.0.0.0 Dialer1
```

## 3. Modern Environment Considerations

- Always keep the MTU at `1492` (or lower, if additional encapsulation
  such as a VPN is stacked on top) on both the Dialer/Virtual-Template
  interfaces. A mismatched MTU is one of the most common causes of
  PPPoE sessions that establish but experience broken web browsing or
  large-transfer stalls (classic "PPPoE MTU black hole" symptoms).
- Pair `ip nat outside` on the client's Dialer interface with `ip tcp
  adjust-mss 1452` on the internal LAN-facing interface if this router
  also serves as a NAT gateway for a LAN — this prevents TCP sessions
  originating behind the router from needing to fragment across the
  reduced-MTU PPPoE link.
- CHAP/PAP secrets are stored in plaintext in `show running-config`
  unless `service password-encryption` (weak, reversible) or a stronger
  Type 6/8 encryption scheme is applied. Treat the running
  configuration as sensitive on both server and client.
- On the server side, if supporting a large number of subscribers,
  consider `virtual-template <n> pre-clone <count>` to pre-build
  Virtual-Access interfaces and reduce session setup latency under
  load — outside the scope of this guide's basic configuration.

> [!IMPORTANT]
> `ppp authentication chap callin` on the client only authenticates
> outbound (client-to-server) direction. If the access concentrator also
> expects the client to challenge it (mutual authentication), omit
> `callin` so the client authenticates in both directions — check with
> your ISP or lab design which behavior is expected.

## 4. Verification and Troubleshooting

### 4.1 Server-Side Verification

```cisco
show pppoe session
```

What it checks and variables to look for:

- **State**: Should show `PTA` (PPP Termination and Aggregation) for
  locally terminated sessions. Any other or missing state means the
  session did not fully establish.
- **SID**: The PPPoE session ID; a unique, nonzero value confirms
  discovery completed successfully for that session.
- **VT / VA**: Confirms the session cloned from the expected Virtual-
  Template and was assigned a Virtual-Access interface.

```cisco
show interface virtual-access 1
```

What it checks and variables to look for:

- **line protocol**: Must be `up`. `down` while the physical interface
  is `up` usually points to a PPP authentication failure.
- **LCP / IPCP state** (visible in `show ppp` variants or `debug ppp
  negotiation` output): Both should show `Open` for a fully negotiated
  session.

```cisco
show ip local pool PPPOE-POOL
```

What it checks and variables to look for:

- **In-use addresses**: Confirms addresses are actually being leased
  out of the expected pool, and that the pool isn't exhausted.

### 4.2 Client-Side Verification

```cisco
show pppoe session
```

What it checks and variables to look for:

- **State**: Should show `PTA` here as well, confirming the client side
  of the session is up.

```cisco
show interface dialer 1
```

What it checks and variables to look for:

- **line protocol**: Must be `up`.
- **Internet address**: Should show the address negotiated from the
  server's pool (e.g., `192.168.100.10/24` from the example in Section
  1.1), confirming `ip address negotiated` succeeded.

```cisco
show ip route
```

What it checks and variables to look for:

- **Default route (`0.0.0.0/0`)**: Should list `Dialer1` as the
  outgoing interface, confirming the static default route from Section
  2.3 is installed and active.

### 4.3 Common Issues

- PPPoE session never forms (no SID assigned): Confirm both the
  physical interface and the dialer pool number match on client and
  server sides (`pppoe-client dial-pool-number` on the client physical
  interface must equal `dialer pool` on the Dialer interface), and that
  both physical interfaces are `no shutdown` and actually passing
  Ethernet frames to each other (check any intermediate switch for
  correct VLAN/port configuration).
- Session forms but immediately drops: Usually a CHAP/PAP authentication
  mismatch. Confirm the `ppp chap hostname`/`ppp chap password` on the
  client exactly match a `username`/`password` pair configured on the
  server (or in RADIUS, if used).
- Session is up but no IP address assigned on the client: Confirm the
  server's `ip local pool` isn't exhausted (Section 4.1) and that
  `peer default ip address pool <name>` on the server's Virtual-
  Template references the correct pool name.
- Session up, address assigned, but browsing is slow or large transfers
  hang: Classic MTU mismatch symptom — confirm `mtu 1492` and `ppp mtu
  adaptive` are configured as shown in Sections 1.2 and 2.2, and that
  `ip tcp adjust-mss` is set if this router performs NAT for a LAN
  (Section 3).

```cisco
debug pppoe events
debug ppp authentication
debug ppp negotiation
```

> [!CAUTION]
> `debug ppp negotiation` and `debug pppoe events` can generate heavy
> log output, particularly on a server aggregating many sessions or
> during a flapping/retrying client. Use conditional debugging where
> supported and disable debugging (`undebug all`) immediately after
> collecting the needed output.
