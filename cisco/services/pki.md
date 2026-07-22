<!-- 
---
title: "Cisco Router Certificate Authority (CA) Server"
author: "Gergő Téringer"
---
 -->

# Cisco Router Certificate Authority (CA) Server

> [!IMPORTANT]
> Modern Context & Compatibility: Operating a Cisco router as a local CA is typically reserved for lab environments, DMVPN testing, or small closed networks. In modern enterprise environments (e.g., Windows 11 clients or Windows Server 2025 PKI infrastructure), it is highly recommended to use a dedicated Enterprise CA (like Microsoft AD CS or a robust Linux PKI solution) for scalability and granular revocation management. Furthermore, this template uses HTTP for SCEP (Simple Certificate Enrollment Protocol); in production, SCEP over HTTPS should be enforced to prevent interception of certificate payloads.

## 1. Server Configuration

The server configuration initializes the local Certificate Authority on the router and starts the HTTP server required to listen for incoming certificate enrollment requests.

```cisco
crypto pki server CA
 database level complete
 no database archive
 grant auto
 cdp-url [http://8.8.8.8/cgi-bin/pkiclient.exe?operation=GetCRL](http://8.8.8.8/cgi-bin/pkiclient.exe?operation=GetCRL)
 no shutdown
 exit
ip http server
```

**Command Breakdown & Explanation:**

- `crypto pki server CA`: Initializes the router's local Certificate Authority service and enters the PKI server configuration mode.
- `database level complete`: Instructs the CA to store complete certificate and request information in its local NVRAM database.
- `no database archive`: Disables the automatic archiving of the CA database to external storage.
- `grant auto`: Automatically grants certificate requests received from clients without requiring manual administrator approval.
- `cdp-url [http://8.8.8.8/cgi-bin/pkiclient.exe?operation=GetCRL](http://8.8.8.8/cgi-bin/pkiclient.exe?operation=GetCRL)`: Defines the Certificate Revocation List (CRL) Distribution Point URL where clients will poll to verify if a peer's certificate has been revoked.
- `no shutdown`: Activates the PKI server service.
- `ip http server`: Enables the local HTTP server, which is mandatory for the CA to listen for SCEP (enrollment) requests and to serve the CRL.

## 2. Client Configuration

The client configuration defines the PKI trustpoint, points it to the CA server for enrollment, and automatically generates the required cryptographic key pairs before fetching and enrolling the certificate.

```cisco
crypto pki trustpoint CA
 enrollment url [http://8.8.8.8:80](http://8.8.8.8:80)
 ip-address 1.1.1.1
 serial-number
 revocation-check crl
 rsakeypair VPN 2048
 auto-enroll 90 regenerate
 exit
crypto pki authenticate CA
crypto pki enroll CA
```

**Command Breakdown & Explanation:**

- `crypto pki trustpoint CA`: Creates a trustpoint on the client router and binds it to the CA.
- `enrollment url [http://8.8.8.8:80](http://8.8.8.8:80)`: Specifies the HTTP URL of the CA server where the client will send its Certificate Signing Request (CSR) via SCEP.
- `ip-address 1.1.1.1`: Includes the specified IP address in the client certificate's Subject Alternative Name (SAN) or Subject field.
- `serial-number`: Instructs the router to include its physical hardware serial number in the certificate request.
- `revocation-check crl`: Forces the client to verify the Certificate Revocation List (CRL) to ensure peer certificates are still valid before trusting them.
- `rsakeypair VPN 2048`: Binds the PKI trustpoint to a specific, locally generated RSA key pair named `VPN` with a 2048-bit length.
- `auto-enroll 90 regenerate`: Configures the router to automatically renew the certificate when `90` percent of its lifetime has expired, generating a new RSA key pair in the process.
- `crypto pki authenticate CA`: Fetches and authenticates the CA's root certificate.
- `crypto pki enroll CA`: Generates the CSR and sends it to the CA for signing.

## 3. Troubleshooting and Verification

### 3.1 Verify CA Server Status

Displays the operational status and database metrics of the local Certificate Authority.

**Command:** `show crypto pki server CA`

**Command Breakdown & Explanation:**
Validates that the CA server is actively running, correctly issuing certificates, and properly tracking the Certificate Revocation List (CRL) for the network.

What it checks and variables to look for:

- **State**: Must be `enabled`
- **Issuer name**: Should match the configured common name or organizational string associated with the CA
- **Granting mode**: Should be `auto`

### 3.2 Verify PKI Trustpoint and Certificates

Displays the installed certificates (both the root CA and the local device certificate) on the client router.

**Command:** `show crypto pki certificates`

**Command Breakdown & Explanation:**
Checks the client router to ensure that the root CA certificate is inherently trusted and that the local device certificate was successfully enrolled, issued, and is currently valid for VPN operations.

What it checks and variables to look for:

- **Status**: Must be `Available` for both the CA and the local router certificates
- **Usage**: Must include `Signature` and `Encryption`
- **Validity Date**: The `end date` must be in the future

<!-- Created by: Gergő Téringer, 2026 -->