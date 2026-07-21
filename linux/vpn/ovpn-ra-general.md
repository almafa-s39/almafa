<!-- 
---
title: "ovpn-ra-general"
author: "Gergő Téringer"
---
 -->
# OpenVPN Remote Access (RA)

This document provides administrative procedures for configuring a Remote Access (RA) Virtual Private Network using OpenVPN on Debian 13 (Trixie). It covers package installation, necessary OpenVPN x509 certificate extensions for server and client trust validation, server configuration, and client profile templates.

> [!NOTE]
> OpenVPN creates a secure virtual private network tunnel using TLS/SSL protocols for key exchange. In a Remote Access topology, roaming clients connect to a central gateway server to access internal networks or route all internet traffic securely.

## 1. Package Installation

Install the core OpenVPN package and easy-rsa utilities to manage certificates and network adapters.

```Bash
# Install OpenVPN daemon and utility packages
apt install openvpn easy-rsa
```

**Command Breakdown & Explanation:**

- `apt install openvpn easy-rsa`: Installs the core OpenVPN routing daemon and certificate generation scripts.

## 2. Required X.509 Certificate Extensions

To prevent Man-in-the-Middle (MITM) attacks and ensure compatibility with modern OpenVPN security flags (`remote-cert-tls server` and `remote-cert-tls client`), certificates must contain precise `extendedKeyUsage` extensions.

### 2.1 Server Certificate Extensions (server.v3.ext)

```Ini, TOML
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = vpn.domain.name
IP.1 = <Server_Public_IP>
```

### 2.2 Client Certificate Extensions (client.v3.ext)

```Ini, TOML
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
```

**Extension Breakdown & Explanation:**

- `extendedKeyUsage = serverAuth`: Mandates that the certificate can only be used to authenticate a TLS server. Essential for the client configuration option `remote-cert-tls server`.
- `extendedKeyUsage = clientAuth`: Mandates that the certificate can only be used to authenticate a TLS client connecting to the server.

## 3. OpenVPN Server Configuration

Create the primary server configuration file at `/etc/openvpn/server/server.conf`. This defines the network pool, encryption ciphers, and routing push directives.

```Bash
# Create the OpenVPN server configuration directory
mkdir -p /etc/openvpn/server

# Write the server configuration profile
cat << 'EOF' > /etc/openvpn/server/server.conf
port 1194
proto udp
dev tun

# Paths to PKI certificates and keys
ca /ca/ca.crt
cert /ca/server.crt
key /ca/server.key
dh /ca/dh.pem

# VPN Subnet Allocation
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Push network routes and DNS to clients (Full/Split Tunnel control)
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"

# General connection options
client-to-client
keepalive 10 60
cipher AES-256-GCM
auth SHA256
user nobody
group nogroup
persist-key
persist-tun

# Logging
status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3
EOF
```

**Command Breakdown & Explanation:**

- `proto udp`: Configures OpenVPN to use UDP on port 1194 for optimal performance and lower latency compared to TCP.
- `server 10.8.0.0 255.255.255.0`: Allocates IP addresses to connecting clients from the specified virtual subnet.
- `push "redirect-gateway def1..."`: Forces clients to route all default gateway traffic through the VPN tunnel (Full Tunnel configuration).

## 4. OpenVPN Client Profile Template (client.ovpn)

The client profile bundles connection parameters and inline cryptographic certificates into a single `.ovpn` file distributed to roaming users.

> [!TIP]
> The `remote-cert-tls server` directive checks the server's certificate against the `serverAuth` extended key usage parameter defined in Section 2.1, providing critical protection against rogue servers.

```Plaintext
clint
dev tun
proto udp
remote <Server_Public_IP> 1194
resolv-retry infinite
nobind
persist-key
persist-tun

# Enforce server certificate verification via extended key usage
remote-cert-tls server

cipher AES-256-GCM
auth SHA256
verb 3

<ca>
# Insert contents of /ca/ca.crt here
</ca>

<cert>
# Insert contents of /ca/client1.crt here
</cert>

<key>
# Insert contents of /ca/client1.key here
</key>
```

## 5. Service Activation and Management

Enable and start the OpenVPN server instance tied to your configuration file name using systemd template units.

```Bash
# Enable and start the OpenVPN server service
systemctl enable openvpn-server@server --now
```

**Command Breakdown & Explanation:**

- `openvpn-server@server`: Systemd template unit that automatically looks for configuration files located in `/etc/openvpn/server/server.conf`.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the OpenVPN daemon status, active listening sockets, and virtual tunnel interface assignment on Debian 13 using standard administrative tools.

### 6.1 Verify OpenVPN service status

**Command:** `systemctl status openvpn-server@server`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/openvpn-server@.service; enabled)`

### 6.2 Verify network listening state on OpenVPN port 1194

**Command:** `ss -tuln | grep 1194`

**What it checks and variables to look for:**

- **State**: Must display `UNCONN` (standard for UDP sockets) or `LISTEN` if running TCP
- **Local Address:Port**: Must display `*:1194` or `0.0.0.0:1194`

### 6.3 Verify virtual TUN network interface creation

**Command:** `ip addr show dev tun0`

**What it checks and variables to look for:**

- **State**: Must be `UP`
- **Inet**: Must display the server's designated gateway tunnel IP (e.g., `10.8.0.1`)

<!-- Created by: Gergő Téringer, 2026 -->