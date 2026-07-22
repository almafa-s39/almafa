<!-- 
---
title: "ProFPTD"
author: "Gergő Téringer"
---
 -->
# ProFTPD

This document provides administrative procedures for installing, securing, and configuring a ProFTPD file server on Debian 13 (Trixie). It details user jailing, login access limits, passive port configuration, and both Explicit TLS (FTPES over port 21) and Implicit TLS (FTPS over port 990) using the `mod_tls` module combined with a central PKI structure.

> [!NOTE]
> ProFTPD is a modular FTP server daemon. Explicit TLS initiates on standard port 21 and upgrades via `AUTH TLS`, whereas Implicit TLS establishes an immediate SSL/TLS wrapper on port 990.

## 1. Package Installation and Core Configuration (proftpd.conf)

Install the required ProFTPD binaries and configure the primary daemon settings, including user jailing, login access control lists, and passive port ranges for firewall traversal.

```Bash
# Install ProFTPD and OpenSSL packages
apt install proftpd-basic openssl
```

Configure the main `/etc/proftpd/proftpd.conf` file to secure user environments and limit authentication access:

- Uncomment `DefaultRoot` to jail users securely into their home directories.
- Enforce PAM user login restrictions using the `<Limit LOGIN>` block.
- Define passive port ranges for data channel routing.
- Include the modular TLS configuration file.

```bash
# Append core configurations to /etc/proftpd/proftpd.conf
cat << 'EOF' >> /etc/proftpd/proftpd.conf

# Jail users into their home directories
DefaultRoot ~

# Restrict login access to specified users
<Limit LOGIN>
  AllowUser webmaster
  DenyAll
</Limit>

# Define passive port range for firewall traversal
PassivePorts 49152 65534

# Include TLS configuration file
Include /etc/proftpd/tls.conf
EOF
```

**Command Breakdown & Explanation:**

- `DefaultRoot ~`: Restricts authenticated users to their respective home directories, preventing directory traversal across the broader system filesystem.
- `<Limit LOGIN>`: Restricts authentication strictly to explicitly permitted accounts (e.g., `webmaster`) and denies all others.
- `PassivePorts`: Allocates unprivileged ephemeral ports for passive data transfers (`PASV`).

## 2. Explicit TLS (FTPES - Port 21) Configuration

Explicit TLS allows clients to connect to standard port 21 unencrypted and subsequently upgrade the session to an encrypted SSL/TLS channel using the `AUTH TLS` command.

> [!TIP]
> Setting `TLSRequired off` permits both standard unencrypted FTP and explicit TLS connections.

```Bash
# Populate /etc/proftpd/tls.conf for Explicit TLS on port 21
cat << 'EOF' > /etc/proftpd/tls.conf
<IfModule mod_tls.c>
  TLSEngine                   on
  TLSProtocol                 TLSv1.3
  TLSRFC2228                  on

  # Allow optional or mandatory TLS upgrades
  TLSRequired                 off
  TLSVerifyClient             off

  # Server certificate, key, and CA certificate paths
  TLSRSACertificateFile       /ca/dmzsrv2/server.crt
  TLSRSACertificateKeyFile    /ca/dmzsrv2/server.key
  TLSCACertificateFile        /ca/CA.crt

  # Logging options
  TLSLog                      /var/log/proftpd/tls.log

  # Options for client compatibility
  TLSOptions                  AllowClientRenegotiation NoCertRequest
</IfModule>
EOF
```

**Command Breakdown & Explanation:**

- `TLSEngine on`: Enables the `mod_tls` module.
- `TLSProtocol TLSv1.3`: Restricts accepted SSL/TLS protocols to modern standards.
- `TLSCACertificateFile`: Defines the trusted Root CA certificate path for validation.
- `TLSVerifyClient off`: Disables mandatory client-side certificate requests during the initial handshake.

## 3. Implicit TLS (FTPS - Port 990) Configuration

Implicit TLS expects an immediate SSL/TLS handshake on port 990 prior to receiving any FTP commands. This is configured via a dedicated VirtualHost block utilizing `TLSOptions UseImplicitSSL`.

> [!WARNING]
> Legacy clients requiring Implicit FTPS must target port 990. Ensure TCP port 990 is permitted in host firewall rules.
> For the next config this guide will create a new configuration file, but you can use `/etc/proftpd/tls.conf` as well, just make sure you import it into your main configuration file!

```Bash
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
    TLSProtocol               TLSv1.3
    
    TLSRSACertificateFile     /ca/dmzsrv2/server.crt
    TLSRSACertificateKeyFile  /ca/dmzsrv2/server.key
    TLSCACertificateFile      /ca/CA.crt
    TLSVerifyClient           off
    
    TLSLog                    /var/log/proftpd/tls_implicit.log
  </IfModule>
</VirtualHost>
EOF

# Restart ProFTPD to apply all configuration changes
systemctl restart proftpd
```

**Command Breakdown & Explanation:**

- `Port 990`: Sets the listening socket to the standard IANA implicit FTPS port.
- `TLSOptions UseImplicitSSL`: Forces `mod_tls` to execute an immediate SSL/TLS wrapper handshake upon TCP connection, bypassing plain-text commands.
- `TLSRequired on`: Disallows unencrypted communication on this virtual server endpoint.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate ProFTPD daemon state, configuration syntax, active listening sockets, and TLS handshakes on Debian 13 using standard administrative tools.

### 4.1 Verify ProFTPD service running status

**Command:** `systemctl status proftpd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/proftpd.service; enabled)`

### 4.2 Verify ProFTPD configuration syntax

**Command:** `proftpd -t`

**What it checks and variables to look for:**

- **Syntax output**: Must display `Syntax check complete.` without throwing configuration parsing errors

### 4.3 Verify network listening state on FTP (21) and FTPS (990) ports

**Command:** `ss -tuln | grep -E ":21|:990"`

**What it checks and variables to look for:**

- **State**: Must display `LISTEN` for both sockets
- **Local Address:Port**: Must list `*:21` (or `0.0.0.0:21`) and `*:990` (or `0.0.0.0:990`)

### 4.4 Verify Explicit TLS handshake on port 21

**Command:** `openssl s_client -starttls ftp -connect 127.0.0.1:21 -showcerts`

**What it checks and variables to look for:**

- **Response Header**: Must display `220` FTP server banner followed by `234 AUTH TLS successful`
- **Verification**: Must complete the TLS handshake and display server certificate details (`BEGIN CERTIFICATE`)

### 4.5 Verify Implicit TLS handshake on port 990

**Command:** `openssl s_client -connect 127.0.0.1:990 -showcerts`

**What it checks and variables to look for:**

- **Handshake**: Must initiate the TLS handshake immediately without sending explicit `starttls` commands
- **Verification**: Must establish an encrypted session and display server certificate details (`BEGIN CERTIFICATE`)

<!-- Created by: Gergő Téringer, 2026 -->