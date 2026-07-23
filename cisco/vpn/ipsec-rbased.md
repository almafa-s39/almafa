<!-- 
---
title: "Cisco IOS Route-Based (VTI) IPsec Site-to-Site VPN: IKEv1 and IKEv2 Configuration Guide"
author: "Gergő Téringer"
---
 -->
# Cisco IOS Route-Based (VTI) IPsec Site-to-Site VPN: IKEv1 and IKEv2 Configuration Guide

## 1. Overview

This document describes how to configure a **route-based** IPsec
Site-to-Site (S2S) VPN on Cisco IOS and IOS XE routers using a Static
Virtual Tunnel Interface (SVTI), for both IKE version 1 (IKEv1) and IKE
version 2 (IKEv2).

Route-based VPNs encrypt traffic based on **routing**, not on an access
list. A logical `Tunnel` interface is created and bound to IPsec via
`tunnel protection ipsec profile`. Any traffic routed into that interface
— by a static route or a dynamic routing protocol such as OSPF or EIGRP —
is encrypted. This removes the need to maintain mirrored crypto ACLs on
both peers and allows dynamic routing to run directly across the tunnel,
which is the main practical advantage over policy-based (crypto map)
designs.

This guide covers the Static VTI (SVTI) model, which is the direct
route-based equivalent of a policy-based point-to-point tunnel. Dynamic
VTI (DVTI), used for hub sites terminating many spoke peers dynamically,
is mentioned briefly in Section 4 but is outside the main scope of this
document.

The commands in this guide apply to Cisco IOS and IOS XE software trains
that support `tunnel mode ipsec ipv4`, `crypto ipsec profile`, and
`crypto ikev2` (IOS 15.x and IOS XE 3.x/16.x/17.x). Always confirm exact
syntax against the Cisco documentation for your specific platform and
software release.

> [!IMPORTANT]
> IKEv1 is a legacy protocol. Cisco and most security frameworks recommend
> IKEv2 for all new deployments. This guide documents IKEv1 for
> interoperability with legacy peers only.

## 2. IKEv1 Route-Based VPN (Static VTI)

### 2.1 ISAKMP (Phase 1) Policy

As with a policy-based design, the ISAKMP policy defines how the two
peers authenticate each other and protect the negotiation channel. The
peers must agree on at least one matching policy before Phase 1 can
complete.

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
!
crypto isakmp key CHANGE-ME-STRONG-PSK address 203.0.113.2
```

**Command Breakdown & Explanation:**

```markdown
- `crypto isakmp policy 10`: Creates ISAKMP policy `10`. The number is a
  priority tag only.
- `encryption aes 256`: Phase 1 encryption algorithm. Avoid `des` and
  `3des`.
- `hash sha256`: Phase 1 integrity/hash algorithm. Avoid `md5` and plain
  `sha` (SHA-1).
- `authentication pre-share`: Uses a pre-shared key for peer
  authentication.
- `group 14`: Diffie-Hellman group 14 (2048-bit) for key exchange. Avoid
  groups 1, 2, and 5.
- `lifetime 86400`: Phase 1 SA lifetime in seconds.
- `crypto isakmp key CHANGE-ME-STRONG-PSK address 203.0.113.2`: Binds a
  pre-shared key to the remote peer's public IP address, matching the
  `tunnel destination` used later.
```

> [!NOTE]
> Unlike a policy-based crypto map design, an SVTI does not require a
> crypto ACL. All traffic routed into the tunnel interface is encrypted
> automatically.

### 2.2 IPsec Transform Set and IPsec Profile

The transform set defines the Phase 2 encryption and integrity
algorithms, exactly as in a policy-based design. In a route-based design
this transform set is attached to a `crypto ipsec profile`, which is
referenced directly by the tunnel interface instead of a crypto map.

```cisco
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-V1
 set transform-set TS-AES256-SHA256
 set pfs group14
```

**Command Breakdown & Explanation:**

- `crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256
  esp-sha256-hmac`: Defines ESP with AES-256 encryption and HMAC-SHA-256
  integrity.
- `mode tunnel`: The default and standard mode for a native (non-GRE)
  IPsec VTI. The virtual tunnel interface itself provides the logical
  point-to-point construct, and IPsec tunnel mode encapsulates the
  original IP packet as usual.
- `crypto ipsec profile IPSEC-PROFILE-V1`: Creates an IPsec profile named
  `IPSEC-PROFILE-V1`. Unlike a crypto map, an IPsec profile has no peer
  address or ACL — those are supplied by the tunnel interface itself.
- `set transform-set TS-AES256-SHA256`: References the transform set
  defined above.
- `set pfs group14`: Enables Perfect Forward Secrecy on Phase 2 rekeys
  using DH group 14.

### 2.3 Tunnel Interface (Static VTI)

The tunnel interface is the core object of a route-based VPN. It has its
own IP address (used for routing across the tunnel), a source and
destination, and a reference to the IPsec profile that protects it.

```cisco
interface Tunnel0
 ip address 172.16.12.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROFILE-V1
!
router ospf 1
 network 172.16.12.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
```

**Command Breakdown & Explanation:**

- `interface Tunnel0`: Creates logical tunnel interface `0`.
- `ip address 172.16.12.1 255.255.255.252`: Assigns a small point-to-point
  transit subnet to the tunnel, used purely for routing adjacency across
  the VPN, not for the protected LAN subnets themselves.
- `tunnel source GigabitEthernet0/0`: The local physical (or logical)
  interface whose IP address is used as the outer/transport source
  address.
- `tunnel destination 203.0.113.2`: The remote peer's public IP address.
  Must match the address used in `crypto isakmp key`.
- `tunnel mode ipsec ipv4`: Declares this a native IPsec VTI (no GRE
  encapsulation), carrying IPv4 payload.
- `tunnel protection ipsec profile IPSEC-PROFILE-V1`: Binds the tunnel
  interface to the IPsec profile, which supplies the transform set,
  PFS setting, and (for IKEv2) the IKEv2 profile.
- `router ospf 1` / `network ... area 0`: Example dynamic routing
  configuration advertising both the tunnel transit subnet and the local
  LAN subnet, so the remote peer learns reachability to
  `10.10.10.0/24` across the tunnel. Static routes pointing at `Tunnel0`
  can be used instead if dynamic routing is not desired.

> [!CAUTION]
> The tunnel interface must be administratively shut down (`shutdown`)
> before applying or changing `tunnel protection ipsec profile`, then
> re-enabled (`no shutdown`) afterward, on most IOS/IOS XE releases.

### 2.4 Full IKEv1 SVTI Configuration Example

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
!
crypto isakmp key CHANGE-ME-STRONG-PSK address 203.0.113.2
!
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-V1
 set transform-set TS-AES256-SHA256
 set pfs group14
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
!
interface Tunnel0
 ip address 172.16.12.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROFILE-V1
!
router ospf 1
 network 172.16.12.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
```

## 3. IKEv2 Route-Based VPN (Static VTI)

IKEv2 uses the same four modular building blocks described for
policy-based deployments — proposal, policy, keyring, and profile — and
attaches to the tunnel interface through the `crypto ipsec profile` via
`set ikev2-profile`, rather than through a crypto map.

### 3.1 IKEv2 Proposal

```cisco
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
```

**Command Breakdown & Explanation:**

- `crypto ikev2 proposal IKEV2-PROPOSAL`: Creates an IKEv2 proposal named
  `IKEV2-PROPOSAL`.
- `encryption aes-cbc-256`: Allowed Phase 1 encryption algorithm(s).
- `integrity sha256`: Allowed integrity/hash algorithm(s).
- `group 14`: Allowed Diffie-Hellman group(s).

> [!NOTE]
> If no proposal is explicitly configured, IOS falls back to a built-in
> `default` proposal that includes weaker legacy algorithms. Always define
> an explicit proposal in production environments.

### 3.2 IKEv2 Policy

```cisco
crypto ikev2 policy IKEV2-POLICY
 match fvrf any
 proposal IKEV2-PROPOSAL
```

**Command Breakdown & Explanation:**

- `crypto ikev2 policy IKEV2-POLICY`: Creates an IKEv2 policy named
  `IKEV2-POLICY`.
- `match fvrf any`: Applies this policy regardless of the front-door VRF
  the negotiation arrives on.
- `proposal IKEV2-PROPOSAL`: References the proposal defined above.

### 3.3 IKEv2 Keyring

```cisco
crypto ikev2 keyring IKEV2-KEYRING
 peer BRANCH-ROUTER
  address 203.0.113.2
  pre-shared-key local CHANGE-ME-LOCAL-KEY
  pre-shared-key remote CHANGE-ME-REMOTE-KEY
```

**Command Breakdown & Explanation:**

- `crypto ikev2 keyring IKEV2-KEYRING`: Creates an IKEv2 keyring.
- `peer BRANCH-ROUTER`: Local descriptive label for this peer entry.
- `address 203.0.113.2`: Matches the remote peer by IP address, which
  must equal the `tunnel destination` configured later.
- `pre-shared-key local CHANGE-ME-LOCAL-KEY`: Key this router sends to
  authenticate itself.
- `pre-shared-key remote CHANGE-ME-REMOTE-KEY`: Key this router expects
  to receive from the peer.

### 3.4 IKEv2 Profile

```cisco
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local IKEV2-KEYRING
 dpd 30 5 on-demand
 no config-exchange request
```

**Command Breakdown & Explanation:**

- `crypto ikev2 profile IKEV2-PROFILE`: Creates the IKEv2 profile.
- `match identity remote address 203.0.113.2 255.255.255.255`: Restricts
  this profile to the specified peer.
- `authentication local pre-share` / `authentication remote pre-share`:
  Both sides authenticate using pre-shared keys.
- `keyring local IKEV2-KEYRING`: References the keyring holding the
  relevant key.
- `dpd 30 5 on-demand`: Dead Peer Detection — probe after 30 seconds of
  inactivity, retry every 5 seconds, only when outbound traffic is
  pending.
- `no config-exchange request`: On VTI deployments, IOS sends an IKEv2
  configuration-request payload by default, which some third-party peers
  do not process correctly and can cause negotiation failures. Disabling
  it avoids this interoperability issue.

> [!WARNING]
> Omitting `no config-exchange request` is a common, hard-to-diagnose
> cause of IKEv2 VTI negotiation failures against non-Cisco peers. If
> Phase 1 completes but the tunnel interface never comes up against a
> third-party device, check this setting first.

### 3.5 IPsec Profile

```cisco
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-V2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set ikev2-profile IKEV2-PROFILE
```

**Command Breakdown & Explanation:**

- `crypto ipsec transform-set TS-AES256-SHA256 ...`: Same Phase 2
  transform-set syntax shared with IKEv1.
- `crypto ipsec profile IPSEC-PROFILE-V2`: Creates the IPsec profile used
  by the IKEv2 tunnel interface.
- `set transform-set TS-AES256-SHA256`: References the transform set.
- `set pfs group14`: Enables PFS on Phase 2 rekeys.
- `set ikev2-profile IKEV2-PROFILE`: This is the key difference from the
  IKEv1 IPsec profile — it explicitly binds the profile to the IKEv2
  profile that determines authentication and identity matching for this
  peer.

### 3.6 Tunnel Interface (Static VTI)

```cisco
interface Tunnel1
 ip address 172.16.13.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROFILE-V2
!
router ospf 1
 network 172.16.13.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
```

**Command Breakdown & Explanation:**

- `interface Tunnel1`: Creates logical tunnel interface `1`, functionally
  identical to the IKEv1 example but using a different interface number
  to avoid conflicting with an existing IKEv1 tunnel on the same router.
- `tunnel protection ipsec profile IPSEC-PROFILE-V2`: References the
  IKEv2-based IPsec profile from Section 3.5.
- All remaining lines (`ip address`, `tunnel source`, `tunnel
  destination`, `tunnel mode ipsec ipv4`, the routing protocol statement)
  function identically to the IKEv1 example in Section 2.3.

### 3.7 Full IKEv2 SVTI Configuration Example

```cisco
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy IKEV2-POLICY
 match fvrf any
 proposal IKEV2-PROPOSAL
!
crypto ikev2 keyring IKEV2-KEYRING
 peer BRANCH-ROUTER
  address 203.0.113.2
  pre-shared-key local CHANGE-ME-LOCAL-KEY
  pre-shared-key remote CHANGE-ME-REMOTE-KEY
!
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local IKEV2-KEYRING
 dpd 30 5 on-demand
 no config-exchange request
!
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-V2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set ikev2-profile IKEV2-PROFILE
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
!
interface Tunnel1
 ip address 172.16.13.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROFILE-V2
!
router ospf 1
 network 172.16.13.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
```

## 4. Design Notes: SVTI, DVTI, and Policy-Based Comparison

This guide focuses on Static VTI (SVTI), the direct route-based
equivalent of a single point-to-point policy-based tunnel. Two related
designs are worth knowing about but are outside this document's scope:

- Dynamic VTI (DVTI): Uses a virtual-template interface that clones a new
  virtual-access interface per incoming peer. Typically used on a hub
  router terminating many spoke peers whose addresses are not known in
  advance.
- DMVPN / FlexVPN: Cisco's scalable hub-and-spoke or full-mesh overlay
  frameworks, built on top of the same underlying VTI/IPsec concepts but
  adding NHRP-based dynamic tunnel discovery.

| Aspect | Policy-Based (Crypto Map) | Route-Based (SVTI) |
| --- | --- | --- |
| Traffic selection | Crypto ACL, mirrored on both peers | Routing (static route or IGP into `Tunnel` interface) |
| Dynamic routing across the tunnel | Not supported | Supported (OSPF, EIGRP, BGP, etc.) |
| Multiple subnets per tunnel | Requires ACL updates on both peers | Automatic, governed by routing |
| Interface model | Applied to a physical interface | Dedicated logical `Tunnel` interface |
| Typical use case | Legacy designs, strict per-subnet control | New deployments, multi-subnet or dynamic-routing environments |

The IKEv1-vs-IKEv2 differences described for policy-based deployments
(negotiation exchanges, NAT-T, asymmetric PSK, DPD tuning) apply equally
here, since both designs share the same underlying IKE negotiation.

## 5. Verification and Troubleshooting

### 5.1 IKEv1 Verification Commands

```cisco
show crypto isakmp sa
```

What it checks and variables to look for:

- **dst / src**: Must match the `tunnel destination` value and the local
  `tunnel source` interface IP.
- **state**: Must be `QM_IDLE` for an established Phase 1 SA. States like
  `MM_KEY_EXCH` or `MM_NO_STATE` indicate a stuck negotiation.

```cisco
show crypto ipsec sa
```

What it checks and variables to look for:

- **local ident / remote ident**: On an SVTI these typically show `0.0.0.0/0.0.0.0`
  (all traffic routed into the interface), rather than specific subnets.
- **pkts encaps / pkts decaps**: Nonzero and increasing values confirm
  bidirectional traffic flow through the tunnel.
- **current outbound spi**: A nonzero SPI confirms Phase 2 has
  successfully negotiated.

```cisco
show interface Tunnel0
```

What it checks and variables to look for:

- **line protocol**: Must be `up`. `line protocol is down` while the
  physical `up` typically indicates the IPsec SA is not established or
  `tunnel destination` is unreachable.
- **MTU**: Confirm it reflects the reduced value accounting for IPsec
  overhead (see Section 5).

### 5.2 IKEv2 Verification Commands

```cisco
show crypto ikev2 sa detailed
```

What it checks and variables to look for:

- **Status**: Must be `READY` for an established Phase 1 SA.
- **Encr / Hash / DH Grp / Auth sign / Auth verify**: Must match the
  algorithms configured in the IKEv2 proposal.
- **Life/Active Time**: Remaining and elapsed SA lifetime.

```cisco
show crypto ikev2 profile
```

What it checks and variables to look for:

- **Profile name**: Must match the profile referenced by `set
  ikev2-profile` in the IPsec profile.
- **Match criteria**: Confirms `match identity remote` is correctly
  scoped to the intended peer.

```cisco
show ip ospf neighbor
```

What it checks and variables to look for:

- **Neighbor ID**: The remote router's OSPF router ID should appear.
- **State**: Should reach `FULL` (or `2WAY` for non-DR/BDR segments) to
  confirm routing is functioning across the tunnel, not just that the
  IPsec SA is up.

### 5.3 Common Issues

- Phase 1 fails immediately: Check basic IP reachability to `tunnel
  destination` and confirm UDP/500 is not blocked.
- Phase 1 completes but the tunnel interface stays down: Confirm the
  IPsec profile is correctly applied via `tunnel protection ipsec
  profile` and that the interface was shut/no-shut after applying it.
- IKEv2 Phase 1 completes but the SVTI never comes up against a
  third-party peer: Check for a missing `no config-exchange request`
  under the IKEv2 profile (Section 3.4).
- Tunnel interface is up but no routes are learned: Check the dynamic
  routing protocol configuration (network statements, area/AS numbers)
  and confirm the tunnel transit subnet is included.
- Intermittent application failures or large-file transfer issues over an
  otherwise stable tunnel: Check tunnel interface MTU and `ip tcp
  adjust-mss` settings (Section 5).

```cisco
debug crypto isakmp
debug crypto ikev2
debug crypto ipsec
debug tunnel protection
```

> [!CAUTION]
> Debug commands can generate heavy CPU load on production routers,
> particularly during active negotiation storms. Use conditional
> debugging (`debug crypto condition peer ipv4 <address>`) where possible
> and disable debugging (`undebug all`) immediately after collecting the
> needed output.

<!-- Created by: Gergő Téringer, 2026 -->