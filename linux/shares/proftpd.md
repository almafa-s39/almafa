<!-- 
---
title: "ProFPTD"
author: "Gergő Téringer"
---
 -->
# ProfTPD

This document provides administrative procedures for installing, securing, and configuring a ProFTPD file server on Debian 13 (Trixie). It details the setup for standard FTP, Explicit TLS (FTPES over port 21), and Implicit TLS (FTPS over port 990) using the `mod_tls` module.

> [!NOTE]
> ProFTPD is a modular FTP server daemon. Explicit TLS (FTPES) initiates on standard port 21 and upgrades the session via `AUTH TLS`, whereas Implicit TLS (FTPS) establishes an SSL/TLS handshake immediately upon connection on port 990.

## 1. Package Installation and SSL Certificate Generation

Install the required ProFTPD binaries and OpenSSL utilities, then generate a dedicated X.509 certificate and private key.

> [!IMPORTANT]
> Ensure certificate key files are protected with strict file permissions (`600`) so unprivileged local accounts cannot read the private key.

```Bash
# Install ProFTPD and OpenSSL packages
apt install proftpd-basic openssl

# Create a dedicated directory for SSL/TLS certificates
mkdir -p /etc/proftpd/ssl

# Generate a self-signed RSA certificate and private key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/proftpd/ssl/proftpd.key.pem \
  -out /etc/proftpd/ssl/proftpd.cert.pem

# Secure private key file permissions
chmod 600 /etc/proftpd/ssl/proftpd.key.pem
chmod 644 /etc/proftpd/ssl/proftpd.cert.pem
```

**Command Breakdown & Explanation:**

- `apt install proftpd-basic openssl`: Installs the core ProFTPD daemon (which includes `mod_tls`) and OpenSSL cryptographic tools.
- `mkdir -p /etc/proftpd/ssl`: Creates a secure folder structure to house server certificates.
- `openssl req -x509...`: Generates a 2048-bit RSA private key and a self-signed certificate valid for 365 days without requiring an interactive passphrase (`-nodes`).
- `chmod 600`: Restricts read and write access on the private key file exclusively to `root`.

## 2. Core Server and Passive Port Configuration

FTP data transfers require secondary TCP connections. Configuring passive port ranges (`PassivePorts`) ensures data channels pass through host firewalls reliably.

```Bash
# Append passive port configuration to /etc/proftpd/proftpd.conf
cat << 'EOF' >> /etc/proftpd/proftpd.conf

# Define passive port range for firewall traversal
PassivePorts 49152 65534

# Enable TLS configuration inclusion
Include /etc/proftpd/tls.conf
EOF
```

**Command Breakdown & Explanation:**

- `PassivePorts 49152 65534`: Directs ProFTPD to allocate unprivileged ephemeral ports between 49152 and 65534 for passive data transfers (`PASV`).
- `Include /etc/proftpd/tls.conf`: Instructs the main configuration parser to include the TLS configuration file.

## 3. Explicit TLS (FTPES - Port 21) Configuration

Explicit TLS allows clients to connect to standard port 21 unencrypted and subsequently upgrade the session to an encrypted SSL/TLS channel using the `AUTH TLS` command.

> [!TIP]
> Setting `TLSRequired off` permits both standard unencrypted FTP and explicit TLS connections. Change to `TLSRequired on` to mandate TLS encryption for all port 21 sessions.

```Bash
# Populate /etc/proftpd/tls.conf for Explicit TLS on port 21
cat << 'EOF' > /etc/proftpd/tls.conf
<IfModule mod_tls.c>
  TLSEngine                   on
  TLSLog                      /var/log/proftpd/tls.log
  TLSProtocol                 TLSv1.2 TLSv1.3
  TLSRFC2228                  on

  # Mandate or allow TLS upgrades (off = optional, on = required)
  TLSRequired                 off

  # Server certificate and key paths
  TLSRSACertificateFile       /etc/proftpd/ssl/proftpd.cert.pem
  TLSRSACertificateKeyFile    /etc/proftpd/ssl/proftpd.key.pem

  # Options for client compatibility
  TLSOptions                  AllowClientRenegotiation NoCertRequest
</IfModule>
EOF
```

**Command Breakdown & Explanation:**

- `TLSEngine on`: Enables the `mod_tls` engine.
- `TLSProtocol TLSv1.2 TLSv1.3`: Restricts accepted SSL/TLS negotiation protocols to modern, secure standards.
- `TLSRFC2228 on`: Enforces RFC 2228 compliance for FTP security extensions.
- `TLSRSACertificateFile` / `TLSRSACertificateKeyFile`: Specifies the local paths to the RSA certificate and private key.
- `TLSOptions NoCertRequest`: Prevents the server from requesting client-side SSL certificates during the handshake.

## 4. Implicit TLS (FTPS - Port 990) Configuration

Implicit TLS expects an immediate SSL/TLS handshake on port 990 prior to receiving any FTP commands. This is configured by creating a dedicated `<VirtualHost>` block using `TLSOptions UseImplicitSSL`.

> [!WARNING]
> Legacy clients requiring Implicit FTPS must target port 990. Ensure TCP port 990 is permitted in host firewall rules.

```bash
# Create a dedicated configuration file for Implicit FTPS on port 990
cat << 'EOF' > /etc/proftpd/conf.d/implicit_ftps.conf
<VirtualHost 0.0.0.0>
  ServerName                  "Implicit FTPS Server"
  Port                        990
  PassivePorts                50000 51000

  <IfModule mod_tls.c>
    TLSEngine                 on
    TLSRequired               on
    TLSOptions                UseImplicitSSL NoCertRequest
    TLSProtocol               TLSv1.2 TLSv1.3
    
    TLSRSACertificateFile     /etc/proftpd/ssl/proftpd.cert.pem
    TLSRSACertificateKeyFile  /etc/proftpd/ssl/proftpd.key.pem
    TLSCACertificateFile      /etc/proftpd/ssl/proftpd.key.pem
    
    TLSLog                    /var/log/proftpd/tls_implicit.log
  </IfModule>
</VirtualHost>
EOF

# Restart ProFTPD to apply all configuration changes
systemctl restart proftpd
```

**Command Breakdown & Explanation:**

- `<VirtualHost 0.0.0.0>`: Binds a separate virtual server instance listening across all available network interfaces.
- `Port 990`: Sets the listening socket to the standard IANA implicit FTPS port 990.
- `TLSOptions UseImplicitSSL`: Forces `mod_tls` to execute an immediate SSL/TLS wrapper handshake upon TCP connection, bypassing the traditional plain-text `AUTH TLS` exchange.
- `TLSRequired on`: Disallows any unencrypted communication on this virtual server endpoint.
- `systemctl restart proftpd`: Reloads the daemon process to bind both port 21 (Explicit) and port 990 (Implicit).

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate ProFTPD daemon state, configuration syntax, active listening sockets, and TLS handshakes on Debian 13 using standard administrative tools.

### 5.1 Verify ProFTPD service running status

**Command:** `systemctl status proftpd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/proftpd.service; enabled)`

### 5.2 Verify ProFTPD configuration syntax

**Command:** `proftpd -t`

**What it checks and variables to look for:**

- **Syntax output**: Must display `Syntax check complete.` without throwing configuration parsing errors

### 5.3 Verify network listening state on FTP (21) and FTPS (990) ports

**Command:** `ss -tuln | grep -E ":21|:990"`

**What it checks and variables to look for:**

- **State**: Must display `LISTEN` for both sockets
- **Local Address:Port**: Must list `*:21` (or `0.0.0.0:21`) and `*:990` (or `0.0.0.0:990`)

### 5.4 Verify Explicit TLS handshake on port 21

**Command:** `openssl s_client -starttls ftp -connect 127.0.0.1:21 -showcerts`

**What it checks and variables to look for:**

- **Response Header**: Must display `220` FTP server banner followed by `234 AUTH TLS successful`
- **Verification**: Must complete the TLS handshake and display server certificate details (`BEGIN CERTIFICATE`)

### 5.5 Verify Implicit TLS handshake on port 990

**Command:** `openssl s_client -connect 127.0.0.1:990 -showcerts`

**What it checks and variables to look for:**

- **Handshake**: Must initiate the TLS handshake immediately without sending explicit `starttls` commands
- **Verification**: Must establish an encrypted session and display server certificate details (`BEGIN CERTIFICATE`)

<!-- Created by: Gergő Téringer, 2026 -->