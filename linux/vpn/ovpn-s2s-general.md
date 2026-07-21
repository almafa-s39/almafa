<!-- 
---
title: "ovpn-s2s-general"
author: "Gergő Téringer"
---
 -->
# OpenVPN Site-to-Site (S2S)

This document provides administrative procedures for establishing a Site-to-Site (S2S) Virtual Private Network using OpenVPN on Debian 13 (Trixie). It utilizes a TLS-based Public Key Infrastructure (PKI) combined with the Client Configuration Directory (CCD) to correctly route subnets between two distinct office networks.

> [!NOTE]
> In an OpenVPN Site-to-Site topology using PKI, the central server must use the `iroute` directive inside a specific client configuration file to understand which internal LAN subnet belongs to which connected VPN peer.

## 1. Package Installation and IP Forwarding

Both the central server (Site A) and the connecting client gateway (Site B) require the OpenVPN daemon and kernel IP packet forwarding to route traffic between their respective LANs.

```Bash
# Install OpenVPN on both gateway nodes
apt install openvpn

# Enable IPv4 packet forwarding in sysctl
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `apt install openvpn`: Installs the core OpenVPN routing daemon.
- `sysctl -p`: Dynamically reloads kernel parameters to permit routing across network interfaces without requiring a reboot.

## 2. Central Server Configuration (Site A)

Assume Site A's local LAN is `10.1.0.0/24` and Site B's local LAN is `10.2.0.0/24`. The server must be instructed via its configuration file to route the `10.2.0.0/24` network into the OpenVPN tunnel adapter.

```Bash
# Create the OpenVPN server and CCD directories
mkdir -p /etc/openvpn/server/ccd

# Write the S2S server configuration profile
cat << 'EOF' > /etc/openvpn/server/server.conf
port 1194
proto udp
dev tun

# PKI Certificates (Generated previously)
ca /ca/ca.crt
cert /ca/server.crt
key /ca/server.key
dh /ca/dh.pem

# VPN Tunnel Subnet
server 10.8.0.0 255.255.255.0

# System routing table path to Site B's LAN via the VPN
route 10.2.0.0 255.255.255.0

# Client Configuration Directory mapping
client-config-dir /etc/openvpn/server/ccd

# Push Site A's LAN route to connecting clients
push "route 10.1.0.0 255.255.255.0"

# General connection options
cipher AES-256-GCM
auth SHA256
keepalive 10 60
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
EOF
```

**Command Breakdown & Explanation:**

- `route 10.2.0.0 255.255.255.0`: Injects a route into the Debian system routing table, directing Site B's LAN traffic to the `tun` interface.
- `client-config-dir /etc/openvpn/server/ccd`: Instructs OpenVPN to look inside this directory for a file matching the connecting client's X.509 Common Name (CN).
- `push "route 10.1.0.0..."`: Automatically configures Site B's routing table to know how to reach Site A's LAN upon connection.

## 3. Client Configuration Directory (CCD) Setup

For OpenVPN to know internally which specific client connection handles the `10.2.0.0/24` subnet, you must create a CCD file named exactly after the client certificate's Common Name.

> [!IMPORTANT]
> The filename created in the `ccd` directory MUST perfectly match the Subject Common Name (CN) embedded in Site B's TLS client certificate (e.g., `site-b-gateway`).

```Bash
# Create the iroute definition file matching the client certificate CN
cat << 'EOF' > /etc/openvpn/server/ccd/site-b-gateway
# Tell the OpenVPN internal routing engine that this specific client owns this subnet
iroute 10.2.0.0 255.255.255.0
EOF
```

**Command Breakdown & Explanation:**

- `iroute 10.2.0.0 255.255.255.0`: Internal Route. While the `route` command in `server.conf` tells the Linux kernel to send packets to the OpenVPN interface, the `iroute` command tells the OpenVPN daemon which specific TLS peer session should receive those packets.

## 4. Client Gateway Configuration (Site B)

On the remote gateway (Site B), create the OpenVPN client configuration. Because this is a permanent infrastructure link, it is typically deployed as a system service rather than a standalone `.ovpn` profile.

```Bash
# Create the client configuration directory structure
mkdir -p /etc/openvpn/client/keys

# Create the client configuration file on Site B
cat << 'EOF' > /etc/openvpn/client/site-b.conf
client
dev tun
proto udp
remote <Site_A_Public_IP> 1194
resolv-retry infinite
nobind
persist-key
persist-tun

# Enforce server certificate verification
remote-cert-tls server

cipher AES-256-GCM
auth SHA256
verb 3

# Direct paths to keys securely transferred to Site B
ca /etc/openvpn/client/keys/ca.crt
cert /etc/openvpn/client/keys/site-b-gateway.crt
key /etc/openvpn/client/keys/site-b-gateway.key
EOF
```

## 5. Service Activation

Start the OpenVPN services on both gateways using their respective systemd unit templates.

```Bash
# On Site A (Central Server)
systemctl enable openvpn-server@server --now

# On Site B (Client Gateway)
systemctl enable openvpn-client@site-b --now
```

**Command Breakdown & Explanation:**

- `openvpn-server@server`: Launches the daemon reading the `/etc/openvpn/server/server.conf` file.
- `openvpn-client@site-b`: Launches the daemon reading the `/etc/openvpn/client/site-b.conf` file.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the CCD mapping, routing tables, and end-to-end connectivity on Debian 13 using standard administrative tools.

### 6.1 Verify OpenVPN routing table injection (Site A)

**Command:** `ip route show dev tun0`

**What it checks and variables to look for:**

- **Output**: Must display the remote LAN subnet (e.g., `10.2.0.0/24`) successfully routed through the active tunnel interface.

### 6.2 Verify internal OpenVPN CCD mapping

**Command:** `cat /var/log/openvpn/openvpn-status.log`

**What it checks and variables to look for:**

- **Routing Table Section**: Must list the client's Common Name (e.g., `site-b-gateway`) mapped directly to the Virtual Tunnel Address and the `10.2.0.0/24` routed subnet.

### 6.3 Verify end-to-end Site-to-Site connectivity

**Command:** `ping -c 4 10.2.0.1` *(executed from Site A)*

**What it checks and variables to look for:**

- **Packet Loss**: Must show `0% packet loss`, indicating successful ICMP routing across the encrypted tunnel and into the remote LAN gateway.

<!-- Created by: Gergő Téringer, 2026 -->