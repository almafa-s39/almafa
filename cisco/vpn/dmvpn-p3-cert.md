<!-- 
---
title: "dmvpn-p3-cert"
author: "Gergő Téringer"
---
 -->
# DMVPN Phase 3 Certificate Authentication

This document provides the standard operating procedures and configuration parameters for deploying a Dynamic Multipoint VPN (DMVPN) Phase 3 environment using IKEv2 and certificate-based (RSA signature) authentication.

> [!IMPORTANT]
> Modern Context & Compatibility: The cryptographic suite utilized in this configuration (AES-GCM-256, SHA-512, and Diffie-Hellman Group 21) aligns with Next-Generation Encryption (NGE) and CNSA standards. This ensures secure deployment on modern Cisco IOS-XE environments (e.g., Cisco Catalyst 8000 series or ASR1000) and integrates safely with modern operating systems without relying on deprecated or vulnerable legacy protocols (like DES, 3DES, or MD5).

[PKI Server and client configuration guide](../services/pki.md)

## 1. Hub Configuration

The Hub router functions as the Next Hop Server (NHS). For DMVPN Phase 3, the Hub must be configured to generate NHRP redirect messages, which instruct Spokes to discover shorter paths and build direct Spoke-to-Spoke IPsec tunnels.

### 1.1 Crypto Configuration

```cisco
crypto pki certificate map CMAP 1
 issuer-name co isp.wsc2022.net
crypto ikev2 proposal IKE-PROP
 encryption aes-gcm-256
 prf sha512
 group 21
crypto ikev2 policy IKE-POL
 match address local 1.1.1.1
 proposal IKE-PROP
crypto ikev2 keyring IKE-KEY
 peer VPN
  address 0.0.0.0 0.0.0.0
crypto ikev2 profile IKE-PROF
 match address local 1.1.1.1
 match certificate CMAP
 authentication remote rsa-sig
 authentication local rsa-sig
 pki trustpoint CA
 no crypto ikev2 http-url cert
crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac
 mode tunnel
crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS
 set ikev2-profile IKE-PROF
```

**Command Breakdown & Explanation:**

- `crypto pki certificate map`: Ensures the Hub only accepts connections from peers presenting a certificate issued by a specific Certificate Authority string (`co isp.wsc2022.net`).
- `crypto ikev2 proposal`: Defines Phase 1 ISAKMP parameters utilizing AES-GCM-256 encryption, SHA-512 for the Pseudo-Random Function, and 521-bit Elliptic Curve (Group 21).
- `crypto ikev2 profile`: Binds local and remote authentication to use RSA signatures (PKI) and links the PKI trustpoint to the interface configuration.
- `crypto ipsec transform-set`: Defines Phase 2 IPSec parameters. Note that `esp-aes 256` combined with `esp-sha512-hmac` ensures robust data-plane security.
- `crypto ipsec profile`: Serves as the logical glue that binds the Phase 1 IKEv2 profile and Phase 2 IPSec transform set to the tunnel interface.

### 1.2 Tunnel Configuration

```cisco
interface Tunnel1
 no shutdown
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 no ip split-horizon eigrp 2022
 ip nhrp network-id 2022
 ip nhrp redirect
 ip summary-address eigrp 2022 192.168.0.0 255.255.0.0
 ip tcp adjust-mss 1360
 tunnel source Loopback0
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC-PROF
```

**Command Breakdown & Explanation:**

- `ip nhrp redirect`: This is the defining command for DMVPN Phase 3 on the Hub. It triggers the Hub to send redirect traffic to spokes, signaling them to dynamically resolve a direct Spoke-to-Spoke path.
- `no ip split-horizon eigrp 2022`: A critical routing prerequisite for Hub-and-Spoke topologies; it allows EIGRP routes learned from one Spoke to be readvertised out the same Tunnel interface to other Spokes.
- `ip mtu 1400` and `ip tcp adjust-mss 1360`: Modifies the Maximum Transmission Unit and TCP Maximum Segment Size to accommodate the GRE and IPSec packet overhead, preventing fragmentation.
- `tunnel mode gre multipoint`: Enables mGRE, allowing a single interface to dynamically establish IPsec connections with multiple endpoints.

## 2. Spoke Configuration

The Spoke configuration is designed to statically register with the Hub (NHS) while remaining capable of interpreting NHRP redirect messages to form dynamic tunnels with other Spokes.

### 2.1 Tunnel Configuration

```cisco
interface Tunnel1
 no shutdown
 ip address 10.0.0.4 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp network-id 2022
 ip nhrp nhs 10.0.0.1 nbma 1.1.1.1 multicast
 ip nhrp nhs 10.0.0.2 nbma 2.2.2.2 multicast
 ip tcp adjust-mss 1360
 tunnel source Loopback0
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC-PROF
```

**Command Breakdown & Explanation:**

- `ip nhrp nhs ... nbma ... multicast`: Statically maps the Hub's overlay tunnel IP to its physical underlay (NBMA) IP, and instructs the router to forward multicast packets (required for EIGRP adjacencies) to the Hub.

> [!TIP]
> In a complete DMVPN Phase 3 Spoke configuration, the command `ip nhrp shortcut` is typically required on the tunnel interface. This allows the Spoke's data plane to dynamically overwrite the routing table's next-hop IP with the newly resolved direct Spoke destination. Consider validating if this is missing from your deployment template.

### 2.2 Crypto Configuration

```cisco
crypto pki certificate map CMAP 1
 issuer-name cn isp.wsc2022.net
crypto ikev2 proposal IKE-PROP
 encryption aes-gcm-256
 prf sha512
 group 21
crypto ikev2 policy IKE-POL
 match address local 1.1.1.1
 proposal IKE-PROP
crypto ikev2 keyring IKE-KEY
 peer VPN
  address 0.0.0.0 0.0.0.0
crypto ikev2 profile IKE-PROF
 match address local 1.1.1.1
 match certificate CMAP
 authentication remote rsa-sig
 authentication local rsa-sig
 pki trustpoint CA
crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha512-hmac
 mode tunnel
crypto ipsec profile IPSEC-PROF
 set transform-set IPSEC-TRANS
 set ikev2-profile IKE-PROF
```

**Command Breakdown & Explanation:**

- The Spoke's cryptographic configurations largely mirror the Hub to ensure parameter agreement during IKEv2 negotiations.
- `issuer-name cn isp.wsc2022.net`: Ensures the Spoke validates the CA identity during PKI certificate exchange before establishing the connection. Note the distinct Common Name (`cn`) matching against the Hub's Configuration (`co`).

## 3. Troubleshooting and Verification

### 3.1 Verify IKEv2 Session

Displays the detailed operational status of Phase 1 IKEv2 negotiations and active sessions.

**Command:** `show crypto ikev2 session detail`

**Command Breakdown & Explanation:**
Validates the Phase 1 IKEv2 status, encryption algorithms, and authentication methods negotiated between the peers to ensure cryptographic parameters match.

What it checks and variables to look for:

- **Status**: Must be `UP-ACTIVE`
- **Encr**: Must be `AES-GCM`
- **Auth**: Must be `rsa-sig`

### 3.2 Verify IPSec Security Associations

Verifies that Phase 2 IPSec Security Associations (SAs) are successfully established and are actively encrypting and decrypting data-plane traffic.

**Command:** `show crypto ipsec sa`

**Command Breakdown & Explanation:**
Checks the active IPSec tunnels to ensure that packets are being successfully encapsulated and decapsulated across the VPN overlay without failure.

What it checks and variables to look for:

- **status**: Must be `active`
- **#pkts encaps**: Must be `> 0` (indicating outbound traffic is being encrypted)
- **#pkts decaps**: Must be `> 0` (indicating inbound traffic is successfully decrypted)

### 3.3 Verify NHRP Registration

Validates NHRP resolution and registration between the Hub and Spokes, which is critical for underlying DMVPN connectivity.

**Command:** `show ip nhrp`

**Command Breakdown & Explanation:**
Checks the Next Hop Resolution Protocol cache to ensure that Spokes have successfully registered their NBMA addresses with the Hub and dynamically discovered other Spokes.

What it checks and variables to look for:

- **Type**: Must be `dynamic` (for Spoke registrations on the Hub) or `static` (for the NHS entry on a Spoke)
- **State**: Must be `up`
- **Flags**: Should include `unique` and `registered`

<!-- Created by: Gergő Téringer, 2026 -->