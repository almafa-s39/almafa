<!-- 
---
title: "WireGuard Site-to-Site VPN"
author: "Gergő Téringer"
---
 -->
# WireGuard Site-to-Site VPN

This document provides administrative procedures for configuring a Site-to-Site (S2S) Virtual Private Network (VPN) using WireGuard on Debian 13 (Trixie). It covers package installation, cryptographic key generation, IP forwarding configuration, and the setup of mutual peer endpoints.

> [!NOTE]
> WireGuard is a modern, high-performance, and lightweight VPN protocol operating in the Linux kernel space. It relies on a peer-to-peer architecture using public-key cryptography, where each node acts identically rather than using a traditional client-server model.

## 1. Package Installation and Key Generation

WireGuard requires the installation of user-space tools to manage the kernel module. After installation, you must generate a private and public key pair for each participating site.

> [!IMPORTANT]
> Private keys must be treated as highly sensitive data. Enforce strict directory permissions (`umask 077`) before generating keys to ensure they are not readable by unprivileged users.

```Bash
# Install the WireGuard tools package
apt install wireguard

# Navigate to the WireGuard configuration directory and enforce secure file creation
cd /etc/wireguard
umask 077

# Generate the private and public key pair for the local site
wg genkey | tee private.key | wg pubkey > public.key
```

**Command Breakdown & Explanation:**

- `apt install wireguard`: Installs the `wg` and `wg-quick` command-line utilities.
- `umask 077`: Ensures any files created subsequently in this shell session will have `-rw-------` permissions (read/write only for `root`).
- `wg genkey`: Generates a base64-encoded Curve25519 private key.
- `wg pubkey`: Derives the corresponding public key from the generated private key via standard input.

## 2. IP Packet Forwarding Configuration

For a Site-to-Site VPN to route traffic between the local subnet and the remote subnet, kernel IP forwarding must be enabled on both WireGuard gateway nodes.

```Bash
# Enable IPv4 packet forwarding in sysctl
echo 'net.ipv4.ip_forward=1/' > /etc/sysctl.d/10-routing.conf

# Create a symbolic link to be able to use the next command
ln -s /etc/sysctl.d/10-routing.conf /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `sysctl -p`: Instructs the kernel to dynamically reload configurations from the `/etc/sysctl.conf` file without requiring a system reboot.

## 3. Site A Configuration

Create the WireGuard interface configuration file (`wg0.conf`) on the first node (Site A). This file defines the local interface parameters and the remote peer (Site B) parameters.

Assume the following topology for this example:

- **Site A LAN**: `10.1.0.0/24`
- **Site B LAN**: `10.2.0.0/24`
- **VPN Tunnel Subnet**: `10.255.255.0/30`

```Bash
# Create and edit the /etc/wireguard/wg0.conf file on Site A
cat << 'EOF' > /etc/wireguard/wg0.conf
[Interface]
# Site A Private Key (contents of /etc/wireguard/private.key on Site A)
PrivateKey = <Insert_Site_A_Private_Key>
Address = 10.255.255.1/30
ListenPort = 51820

[Peer]
# Site B Public Key
PublicKey = <Insert_Site_B_Public_Key>
# Site B Public WAN IP and Port
Endpoint = <Site_B_Public_IP>:51820
# Permitted IPs to route through this tunnel (Site B Tunnel IP + Site B LAN)
AllowedIPs = 10.255.255.2/32, 10.2.0.0/24
EOF
```

**Command Breakdown & Explanation:**

- `[Interface]`: Defines the settings for the local `wg0` network adapter.
- `ListenPort`: The UDP port the WireGuard kernel module will bind to (default is 51820). Ensure this port is permitted through your external firewall.
- `[Peer]`: Defines the remote endpoint attempting to connect or be connected to.
- `Endpoint`: The reachable external IP address and port of the remote site.
- `AllowedIPs`: Dictates which destination IP subnets will be routed across the VPN interface, and acts as a strict ingress filter for incoming packets from this peer.

## 4. Site B Configuration

Create the corresponding configuration file on the second node (Site B), mirroring the logic applied to Site A.

> [!TIP]
> If one of the sites has a dynamic IP or is behind a NAT, you can omit the `Endpoint` directive on that side. WireGuard will automatically learn the endpoint address once the NAT'd peer initiates the handshake.

```Bash
# Create and edit the /etc/wireguard/wg0.conf file on Site B
cat << 'EOF' > /etc/wireguard/wg0.conf
[Interface]
# Site B Private Key (contents of /etc/wireguard/private.key on Site B)
PrivateKey = <Insert_Site_B_Private_Key>
Address = 10.255.255.2/30
ListenPort = 51820

[Peer]
# Site A Public Key
PublicKey = <Insert_Site_A_Public_Key>
# Site A Public WAN IP and Port
Endpoint = <Site_A_Public_IP>:51820
# Permitted IPs to route through this tunnel (Site A Tunnel IP + Site A LAN)
AllowedIPs = 10.255.255.1/32, 10.1.0.0/24
EOF
```

## 5. Service Activation

Once the configurations are established and keys are swapped securely, you can start the WireGuard interfaces using the `wg-quick` service wrapper.

```Bash
# Enable the WireGuard interface to start automatically on boot and launch it now
systemctl enable wg-quick@wg0 --now
```

**Command Breakdown & Explanation:**

- `wg-quick@wg0`: Reads the `/etc/wireguard/wg0.conf` file, dynamically creates the `wg0` network interface, assigns the IP addresses, configures the cryptographic routing, and injects the necessary routes into the system routing table based on the `AllowedIPs` directive.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the interface status, cryptographic handshakes, and routing tables on Debian 13 using native WireGuard and IP commands.

### 6.1 Verify WireGuard interface status and handshakes

**Command:** `wg show`

**What it checks and variables to look for:**

- **interface**: Must display `wg0` with the correct listening port
- **peer**: Must list the Public Key of the remote site
- **endpoint**: Must display the currently resolved external IP and port of the peer
- **latest handshake**: Must show a recent time duration (e.g., `1 minute, 15 seconds ago`). If this is missing or empty, the cryptographic handshake has failed or UDP packets are being blocked by a firewall.

### 6.2 Verify wg-quick systemd service status

**Command:** `systemctl status wg-quick@wg0`

**What it checks and variables to look for:**

- **Active**: Must be `active (exited)` (expected behavior for interface setup wrappers)
- **Loaded**: Must be `loaded (/lib/systemd/system/wg-quick@.service; enabled)`

### 6.3 Verify routing table injections

**Command:** `ip route show dev wg0`

**What it checks and variables to look for:**

- **Output**: Must display the exact subnets declared in your `AllowedIPs` configuration (e.g., `10.2.0.0/24 scope link`)

### 6.4 Verify end-to-end tunnel connectivity

**Command:** `ping -c 4 10.255.255.2` *(executed from Site A)*

**What it checks and variables to look for:**

- **Packet Loss**: Must show `0% packet loss`, indicating successful ICMP

<!-- Created by: Gergő Téringer, 2026 -->