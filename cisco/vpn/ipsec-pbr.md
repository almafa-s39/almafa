<!-- 
---
title: "Cisco IOS Policy-Based IPsec Site-to-Site VPN: IKEv1 and IKEv2 Configuration Guide"
author: "Gergő Téringer"
---
 -->
# Cisco IOS Policy-Based IPsec Site-to-Site VPN: IKEv1 and IKEv2 Configuration Guide

## 1. Overview

This document describes how to configure a **policy-based** (crypto map based)
IPsec Site-to-Site (S2S) VPN on Cisco IOS and IOS XE routers using both
IKE version 1 (IKEv1) and IKE version 2 (IKEv2).

Policy-based VPNs use an extended access list ("interesting traffic" or
crypto ACL) to define which traffic is encrypted, as opposed to route-based
VPNs (VTI - Virtual Tunnel Interface), which encrypt everything routed into
a tunnel interface. Policy-based VPNs remain common on older platforms,
in environments with strict per-subnet encryption requirements, or where
route-based (VTI) designs are not available.

Both IKEv1 and IKEv2 use the same two-phase model:

- Phase 1 negotiates a secure, authenticated channel between peers (the
  IKE/ISAKMP Security Association).
- Phase 2 negotiates the actual IPsec Security Associations that protect
  user traffic.

The commands in this guide apply to Cisco IOS and IOS XE software trains
that support `crypto ikev2` and `crypto isakmp` command sets (IOS 15.x and
IOS XE 3.x/15.x/17.x). Always confirm exact syntax against the Cisco
documentation for your specific platform and software release, since minor
keyword differences exist between platforms (ISR, ASR) and code trains.

> [!IMPORTANT]
> IKEv1 is a legacy protocol. Cisco and most security frameworks recommend
> IKEv2 for all new deployments. This guide documents IKEv1 for
> interoperability with legacy peers only.

## 2. IKEv1 Policy-Based IPsec VPN

### 2.1 ISAKMP (Phase 1) Policy

The ISAKMP policy defines how the two peers authenticate each other and
protect the negotiation channel itself. Cisco IOS evaluates ISAKMP
policies in priority order (lowest number = highest priority) and the two
peers must agree on at least one matching policy (encryption, hash,
authentication method, Diffie-Hellman group, and lifetime) before Phase 1
can complete.

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

- `crypto isakmp policy 10`: Creates (or edits) ISAKMP policy number `10`.
  The number is a priority tag only, not a strength indicator.
- `encryption aes 256`: Sets the Phase 1 encryption algorithm to AES with a
  256-bit key. Avoid `des` and `3des`, which are cryptographically weak.
- `hash sha256`: Sets the Phase 1 integrity/hash algorithm. Avoid `md5` and
  plain `sha` (SHA-1), both considered weak for new deployments.
- `authentication pre-share`: Uses a pre-shared key (PSK) for peer
  authentication. Certificate-based authentication (`rsa-sig`) is also
  supported but outside the scope of this document.
- `group 14`: Selects Diffie-Hellman group 14 (2048-bit MODP) for key
  exchange. Groups 1, 2, and 5 are considered weak and should not be used.
- `lifetime 86400`: Sets the Phase 1 SA lifetime in seconds (24 hours is
  the Cisco default).
- `crypto isakmp key CHANGE-ME-STRONG-PSK address 203.0.113.2`: Binds a
  pre-shared key to the remote peer's public IP address. Both peers must
  configure the identical key value.

> [!WARNING]
> Pre-shared keys configured in plaintext are visible in `show running-config`
> unless password encryption (`service password-encryption` or a stronger
> Type 6/8 encryption scheme) is applied. Treat the running configuration as
> sensitive.

### 2.2 IPsec Transform Set and Crypto ACL (Phase 2)

The transform set defines the encryption and integrity algorithms used to
protect actual data traffic in Phase 2. The crypto access list (ACL)
defines the "interesting traffic" that triggers and is protected by the
tunnel. This ACL must be mirrored (source/destination reversed) on the
remote peer.

```cisco
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended VPN-TRAFFIC
 permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
```

**Command Breakdown & Explanation:**

- `crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac`:
  Defines a transform set named `TS-AES256-SHA256` using ESP with AES-256
  encryption and HMAC-SHA-256 integrity.
- `mode tunnel`: Encapsulates the entire original IP packet, which is
  required for site-to-site VPNs connecting two private networks. `mode
  transport` is used only for host-to-host scenarios.
- `ip access-list extended VPN-TRAFFIC`: Creates a named extended ACL that
  identifies interesting traffic.
- `permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255`: Matches traffic
  from the local `10.10.10.0/24` subnet to the remote `10.20.20.0/24`
  subnet. The remote peer must configure the mirrored statement (source
  and destination swapped).

> [!NOTE]
> Crypto ACLs use wildcard masks, not subnet masks. `0.0.0.255` matches a
> `/24` network.

### 2.3 Crypto Map and Interface Application

The crypto map ties together the peer address, the transform set, the
crypto ACL, and (optionally) Perfect Forward Secrecy (PFS). The crypto map
is then applied to the outbound (typically WAN-facing) interface.

```cisco
crypto map CMAP-IKEV1 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set security-association lifetime seconds 3600
 match address VPN-TRAFFIC
!
interface GigabitEthernet0/0
 crypto map CMAP-IKEV1
```

**Command Breakdown & Explanation:**

- `crypto map CMAP-IKEV1 10 ipsec-isakmp`: Creates crypto map `CMAP-IKEV1`
  entry `10`, using IKE-based (`ipsec-isakmp`) negotiation rather than
  manually keyed SAs.
- `set peer 203.0.113.2`: Defines the remote VPN peer's public IP address.
- `set transform-set TS-AES256-SHA256`: References the Phase 2 transform
  set defined earlier.
- `set pfs group14`: Enables Perfect Forward Secrecy using DH group 14 for
  Phase 2 rekeys, ensuring a compromised long-term key cannot be used to
  derive past session keys.
- `set security-association lifetime seconds 3600`: Overrides the default
  Phase 2 SA lifetime (default is 3,600 seconds / 1 hour).
- `match address VPN-TRAFFIC`: Associates the crypto ACL with this crypto
  map entry.
- `crypto map CMAP-IKEV1` (under the interface): Applies the crypto map to
  the physical or logical interface facing the VPN peer.

> [!CAUTION]
> Only one crypto map set can be applied to a given interface. If multiple
> peers or tunnels are required, add additional sequence numbers to the
> same crypto map name rather than applying a second crypto map.

### 2.4 Full IKEv1 Configuration Example

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
ip access-list extended VPN-TRAFFIC
 permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
!
crypto map CMAP-IKEV1 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set security-association lifetime seconds 3600
 match address VPN-TRAFFIC
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 crypto map CMAP-IKEV1
```

## 3. IKEv2 Policy-Based IPsec VPN

IKEv2 replaces the single monolithic ISAKMP policy with four discrete,
modular building blocks: proposal, policy, keyring, and profile. This
separation allows more granular control, faster negotiation (fewer
message exchanges than IKEv1), native NAT traversal, and built-in support
for asymmetric pre-shared keys and Dead Peer Detection (DPD) tuning.

### 3.1 IKEv2 Proposal

An IKEv2 proposal is the equivalent of the algorithm portion of an IKEv1
ISAKMP policy. Unlike IKEv1, a single IKEv2 proposal can list multiple
acceptable algorithms, letting the router offer several options in one
object.

```cisco
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
```

**Command Breakdown & Explanation:**

- `crypto ikev2 proposal IKEV2-PROPOSAL`: Creates an IKEv2 proposal named
  `IKEV2-PROPOSAL`.
- `encryption aes-cbc-256`: Sets the allowed Phase 1 encryption
  algorithm(s). Multiple values can be listed on one line to offer several
  options (e.g., `aes-cbc-256 aes-cbc-128`).
- `integrity sha256`: Sets the allowed integrity/hash algorithm(s).
- `group 14`: Sets the allowed Diffie-Hellman group(s) for key exchange.

> [!NOTE]
> If no proposal is explicitly configured, IOS uses a built-in `default`
> IKEv2 proposal containing a broad set of algorithms, including weaker
> legacy options such as 3DES and DH group 2. Always define an explicit
> proposal in production environments to enforce strong algorithms only.

### 3.2 IKEv2 Policy

An IKEv2 policy links one or more proposals to a given local address or
front-door VRF (FVRF). Most single-VRF deployments use a single policy
referencing a single proposal.

```cisco
crypto ikev2 policy IKEV2-POLICY
 match fvrf any
 proposal IKEV2-PROPOSAL
```

**Command Breakdown & Explanation:**

- `crypto ikev2 policy IKEV2-POLICY`: Creates an IKEv2 policy named
  `IKEV2-POLICY`.
- `match fvrf any`: Applies this policy regardless of the front-door VRF
  the negotiation arrives on. This line is optional in non-VRF designs
  but is commonly included for clarity.
- `proposal IKEV2-PROPOSAL`: References the proposal defined in the
  previous step. A policy can reference multiple proposals; the peers
  negotiate the best mutually supported combination.

### 3.3 IKEv2 Keyring

The IKEv2 keyring stores pre-shared keys per peer, independently from
IKEv1's global `crypto isakmp key` command. IKEv2 keyrings additionally
support asymmetric keys (a different key for local vs. remote
authentication).

```cisco
crypto ikev2 keyring IKEV2-KEYRING
 peer BRANCH-ROUTER
  address 203.0.113.2
  pre-shared-key local CHANGE-ME-LOCAL-KEY
  pre-shared-key remote CHANGE-ME-REMOTE-KEY
```

**Command Breakdown & Explanation:**

- `crypto ikev2 keyring IKEV2-KEYRING`: Creates an IKEv2 keyring named
  `IKEV2-KEYRING`.
- `peer BRANCH-ROUTER`: Creates a peer entry with the descriptive label
  `BRANCH-ROUTER` (label is local-significant only).
- `address 203.0.113.2`: Matches the remote peer by IP address. A subnet,
  FQDN, or range can also be used depending on IOS version.
- `pre-shared-key local CHANGE-ME-LOCAL-KEY`: Key this router sends to
  authenticate itself to the peer.
- `pre-shared-key remote CHANGE-ME-REMOTE-KEY`: Key this router expects to
  receive from the peer. In most designs the local and remote key values
  are set identically on both ends (symmetric), but IKEv2 supports
  distinct values if desired.

> [!TIP]
> Using identical local/remote key values on both peers is the simplest
> and most common configuration. Reserve asymmetric keys for scenarios
> with specific compliance or key-rotation requirements.

### 3.4 IKEv2 Profile

The IKEv2 profile is the central object that ties together peer identity
matching, authentication method, and the keyring. It is referenced later
by the crypto map (or IPsec profile, in route-based designs).

```cisco
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local IKEV2-KEYRING
 dpd 30 5 on-demand
```

**Command Breakdown & Explanation:**

- `crypto ikev2 profile IKEV2-PROFILE`: Creates an IKEv2 profile named
  `IKEV2-PROFILE`.
- `match identity remote address 203.0.113.2 255.255.255.255`: Restricts
  this profile to peers identifying with the exact address
  `203.0.113.2`. `match identity remote any` can be used for
  dynamic-peer designs.
- `authentication local pre-share`: This router authenticates itself using
  a pre-shared key.
- `authentication remote pre-share`: This router expects the peer to
  authenticate using a pre-shared key.
- `keyring local IKEV2-KEYRING`: References the keyring holding the
  relevant pre-shared key(s).
- `dpd 30 5 on-demand`: Enables Dead Peer Detection; probes the peer after
  30 seconds of inactivity, retries every 5 seconds if unanswered, and
  only sends probes when there is outbound traffic pending
  (`on-demand`), rather than continuously (`periodic`).

### 3.5 IPsec Transform Set and Crypto Map (Policy-Based)

For a policy-based (crypto map) deployment, IKEv2 reuses the same
`crypto ipsec transform-set` and crypto ACL constructs as IKEv1. The
crypto map entry additionally references the IKEv2 profile via
`set ikev2-profile`.

```cisco
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended VPN-TRAFFIC-V2
 permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
!
crypto map CMAP-IKEV2 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC-V2
!
interface GigabitEthernet0/0
 crypto map CMAP-IKEV2
```

**Command Breakdown & Explanation:**

- `crypto ipsec transform-set TS-AES256-SHA256 ...`: Same Phase 2
  transform-set syntax as IKEv1; IKEv1 and IKEv2 share this construct.
- `ip access-list extended VPN-TRAFFIC-V2`: Defines interesting traffic,
  mirrored on the remote peer, exactly as in the IKEv1 example.
- `crypto map CMAP-IKEV2 10 ipsec-isakmp`: Creates the crypto map entry.
  The `ipsec-isakmp` keyword is retained for backward compatibility even
  though the negotiation actually uses IKEv2.
- `set ikev2-profile IKEV2-PROFILE`: This is the key difference from the
  IKEv1 crypto map — it explicitly binds the map entry to the IKEv2
  profile, which in turn determines the authentication and identity
  matching used for this peer.
- All remaining lines (`set peer`, `set transform-set`, `set pfs`,
  `match address`) function identically to the IKEv1 example.

> [!IMPORTANT]
> A crypto map entry must reference either an IKEv1-style ISAKMP policy
> (implicitly, via global config) or an explicit `set ikev2-profile`, not
> both simultaneously for the same peer/entry. Mixing IKEv1 and IKEv2
> peers under different sequence numbers of the same crypto map name is
> supported.

### 3.6 Full IKEv2 Configuration Example

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
!
crypto ipsec transform-set TS-AES256-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended VPN-TRAFFIC-V2
 permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
!
crypto map CMAP-IKEV2 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS-AES256-SHA256
 set pfs group14
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC-V2
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 crypto map CMAP-IKEV2
```

## 4. IKEv1 vs IKEv2 Comparison

The table below summarizes the practical differences relevant to a
policy-based S2S VPN design.

| Aspect | IKEv1 | IKEv2 |
| --- | --- | --- |
| Configuration objects | Single ISAKMP policy | Proposal, policy, keyring, profile |
| Negotiation exchanges | 6 messages (Main Mode) or 3 (Aggressive Mode) | 4 messages |
| NAT traversal | Supported, less consistently implemented | Native, standardized |
| Asymmetric PSK | Not supported | Supported |
| Built-in DPD tuning | Manual, separate command | Integrated into the profile |
| Multiple algorithm sets per object | No (one policy = one algorithm set) | Yes (one proposal can list several) |
| Cisco/industry guidance | Legacy, interoperability only | Recommended for new deployments |

## 5. Verification and Troubleshooting

### 5.1 IKEv1 Verification Commands

```cisco
show crypto isakmp sa
```

What it checks and variables to look for:

- **dst / src**: Destination and source IP addresses. Must match the
  `set peer` value and the local outbound interface IP.
- **state**: Phase 1 negotiation state. Must be `QM_IDLE` for an
  established tunnel. States like `MM_KEY_EXCH` or `MM_NO_STATE`
  indicate the negotiation is stuck.
- **conn-id**: A unique local identifier for the SA; useful for cross
  referencing with debug output.

```cisco
show crypto ipsec sa
```

What it checks and variables to look for:

- **local ident / remote ident**: The subnets from the crypto ACL. Must
  match the mirrored ACL entries on both peers.
- **pkts encaps / pkts decaps**: Packet counters. Both values greater
  than `0` and increasing confirms bidirectional traffic flow through
  the tunnel.
- **PERMIT / current outbound spi**: A non-zero SPI value confirms Phase
  2 has successfully negotiated.

### 5.2 IKEv2 Verification Commands

```cisco
show crypto ikev2 sa detailed
```

What it checks and variables to look for:

- **Status**: Must be `READY` for an established Phase 1 SA.
- **Encr / Hash / DH Grp / Auth sign / Auth verify**: Must show the
  algorithms configured in the IKEv2 proposal (e.g., `AES-CBC`,
  `SHA256`, `14`).
- **Life/Active Time**: Remaining and elapsed SA lifetime; useful for
  confirming rekey timing.

```cisco
show crypto ikev2 profile
```

What it checks and variables to look for:

- **Profile name**: Must match the profile referenced by `set
  ikev2-profile` in the crypto map.
- **Match criteria**: Confirms the `match identity remote` value is
  correctly scoped to the intended peer.
- **Keyring**: Confirms the correct keyring name is associated with the
  profile.

```cisco
show crypto ipsec sa
```

What it checks and variables to look for:

- **pkts encaps / pkts decaps**: Same interpretation as the IKEv1
  section; nonzero and increasing values confirm active traffic.
- **inbound / outbound spi**: Both should be populated with nonzero
  hexadecimal values.

### 5.3 Common Issues

- Phase 1 fails immediately, no debug output: Check basic IP reachability
  and confirm UDP/500 is not blocked between peers.
- Phase 1 stuck in a partial state: Confirm the ISAKMP policy (IKEv1) or
  proposal (IKEv2) matches exactly on both peers — encryption, hash/
  integrity, DH group, authentication method, and lifetime.
- Phase 1 succeeds but Phase 2 never comes up: Confirm the crypto ACLs
  are exact mirror images of each other and that the transform sets
  match on both peers.
- Tunnel is up but no traffic passes: Check routing (a route to the
  remote subnet must point toward the crypto-map-enabled interface) and
  confirm the interesting traffic ACL is not too narrow.

```cisco
debug crypto isakmp
debug crypto ikev2
debug crypto ipsec
```

> [!CAUTION]
> Debug commands can generate heavy CPU load on production routers,
> particularly during active negotiation storms. Use conditional
> debugging (`debug crypto condition peer ipv4 <address>`) where possible
> and disable debugging (`undebug all`) immediately after collecting the
> needed output.

<!-- Created by: Gergő Téringer, 2026 -->