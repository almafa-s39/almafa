<!-- 
---
title: "Cisco IOS AnyConnect SSL VPN"
author: "Gergő Téringer"
---
 -->
# Cisco IOS AnyConnect SSL VPN

```markdown
This document details the configuration for deploying an AnyConnect Secure Mobility Client (SSL VPN / WebVPN) directly on a Cisco IOS router. It covers the necessary prerequisites, AAA authentication, PKI certificate management, and the WebVPN gateway/context configuration.

> [!WARNING]
> Modern Context & Compatibility: Configuring AnyConnect WebVPN directly on Cisco IOS routers is a legacy approach and has been largely deprecated in modern Cisco IOS-XE environments in favor of dedicated Cisco Secure Firewall (FTD) or ASA appliances. Additionally, older AnyConnect `.pkg` files may fail to install or establish secure connections on modern operating systems like Windows 11 due to deprecated cryptographic ciphers (e.g., TLS 1.0/1.1). Always ensure you are deploying a modern Cisco Secure Client (formerly AnyConnect) package that supports TLS 1.2 or TLS 1.3.
```

## 1. Prerequisites and File Management

Before configuring the VPN, the router must have an RSA key pair, the AnyConnect client package (`.pkg`), and the PKCS12 certificate bundle (`.pfx`) stored in its local flash memory.

```cisco
crypto key generate rsa modulus 2048
copy tftp: flash:anyconnect.pkg
copy tftp: flash:hqfw.pfx
crypto vpn anyconnect flash:anyconnect.pkg
```

**Command Breakdown & Explanation:**

- `crypto key generate rsa`: Generates the RSA key pair required for SSL/TLS negotiations (assuming keys weren't previously generated).
- `copy tftp: flash:`: Demonstrates retrieving the AnyConnect client package and the certificate bundle from an external server to the router's local flash.
- `crypto vpn anyconnect`: Defines the location of the AnyConnect client package on the local flash so the router can push it to client machines connecting via the web portal.

## 2. AAA and User Authentication

Defines the local authentication database and creates the necessary AAA lists to authenticate connecting VPN users.

```cisco
aaa new-model
aaa authentication login VPN_AUTH local
username vpnuser privilege 0 password Passw0rd!
```

**Command Breakdown & Explanation:**

- `aaa new-model`: Enables the Authentication, Authorization, and Accounting (AAA) architecture on the router.
- `aaa authentication login VPN_AUTH local`: Creates a custom login authentication list named `VPN_AUTH` that points to the router's local database.
- `username`: Creates a local user (`vpnuser`) with a password.

> [!TIP]
> Explicitly setting `privilege 0` is a good security practice for VPN users stored in the local database to ensure they cannot access the router's EXEC mode if they attempt to SSH into the device.

## 3. PKI and Certificate Import

Imports the PKCS12 certificate bundle (which contains both the public certificate and the private key) to be used by the SSL VPN Gateway.

```cisco
crypto pki trustpoint CASrv
 enrollment pkcs12
 revocation-check none
 exit
crypto pki import CASrv pkcs12 flash:hqfw.pfx password Passw0rd
```

**Command Breakdown & Explanation:**

- `crypto pki trustpoint CASrv`: Creates a trustpoint named `CASrv` to manage the SSL certificate.
- `enrollment pkcs12`: Specifies that the certificate will be enrolled (imported) using a PKCS12 formatted file, rather than SCEP or manual terminal pasting.
- `revocation-check none`: Disables CRL/OCSP checking for this specific trustpoint.
- `crypto pki import`: Executes the import of the `.pfx` file from the flash memory, using the extraction password (`Passw0rd`) defined when the bundle was created.

## 4. WebVPN Gateway and Context Configuration

This section provisions the actual SSL VPN gateway (the interface and port listening for connections) and the context (the policies, IP pools, and routing injected into the client).

```cisco
ip local pool VPN-POOL 100.100.100.1 100.100.100.10

webvpn gateway VPN-GW
 ip address 172.20.4.1 port 443
 ssl trustpoint CASrv
 inservice
 exit

webvpn context CONTEXT
 policy group VPN-GP
  functions svc-enabled
  svc address-pool VPN-POOL netmask 255.255.255.0
  svc rekey method new-tunnel
  svc split include 172.16.0.0 255.240.0.0
  svc split include 1.1.1.1 255.255.255.255
  exit
 default-group policy VPN-GP
 aaa authentication list VPN_AUTH
 gateway VPN-GW
 max-users 10
 inservice
 exit

ip http server
ip http secure-server
```

**Command Breakdown & Explanation:**

- `ip local pool`: Defines the range of IP addresses (`100.100.100.1` to `.10`) that will be dynamically assigned to AnyConnect clients.
- `webvpn gateway`: Creates the listener for incoming SSL VPN connections.
  - `ip address ... port 443`: Binds the listener to a specific IP address and standard HTTPS port.
  - `ssl trustpoint`: Attaches the imported certificate (`CASrv`) to secure the HTTPS tunnel.
  - `inservice`: Activates the gateway.
- `webvpn context`: Defines the logical SSL VPN instance.
  - `functions svc-enabled`: Enables the Secure VPN Client (SVC / AnyConnect) functionality.
  - `svc address-pool`: Links the previously created IP pool to the client policy.
  - `svc split include`: Configures split tunneling, dictating that only traffic destined for `172.16.0.0/12` and `1.1.1.1/32` will traverse the VPN tunnel; all other traffic uses the client's local internet connection.
- `aaa authentication list`: Binds the AAA list `VPN_AUTH` to the context to enforce user authentication.
- `ip http server` / `ip http secure-server`: Mandates that the router's HTTP/HTTPS engines are running, as WebVPN relies on these underlying processes to serve the AnyConnect portal.

## 5. Troubleshooting and Verification

### 5.1 Verify WebVPN Gateway Status

Checks the operational state of the WebVPN gateway to ensure it is actively listening on the correct IP and port.

**Command:** `show webvpn gateway`

**Command Breakdown & Explanation:**
Validates the SSL listener and ensures the correct SSL trustpoint is successfully bound without cryptographic errors.

What it checks and variables to look for:

- **Admin Status**: Must be `up`
- **Operation Status**: Must be `up`

### 5.2 Verify WebVPN Context Configuration

Displays the specific settings, assigned AAA lists, and policies for the configured WebVPN context.

**Command:** `show webvpn context`

**Command Breakdown & Explanation:**
Ensures that the AnyConnect context is active and that the split-tunneling policies and address pools are correctly applied to the instance.

What it checks and variables to look for:

- **Status**: Must be `in service`
- **Default Group Policy**: Must match `VPN-GP`
- **AAA Authentication List**: Must match `VPN_AUTH`

### 5.3 Verify Active AnyConnect Sessions

Displays real-time information about users currently connected to the SSL VPN.

**Command:** `show webvpn session context all`

**Command Breakdown & Explanation:**
Validates that remote users are successfully authenticating and receiving IP addresses from the `VPN-POOL`.

What it checks and variables to look for:

- **Username**: Should match the authenticated user (e.g., `vpnuser`).
- **Session Type**: Must include `SVC` or `AnyConnect` (indicating a full tunnel client, not a clientless portal session).
- **Client IP**: Must be an IP from the `100.100.100.x` range.

<!-- Created by: Gergő Téringer, 2026 -->