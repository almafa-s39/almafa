<!-- 
---
title: "wireguard-ra"
author: "Gergő Téringer"
---
 -->
# WireGuard Remote Access (RA) VPN

This document provides administrative procedures for configuring a Remote Access (RA) Virtual Private Network (VPN) using WireGuard on Debian 13 (Trixie). Unlike Site-to-Site connections, a Remote Access VPN allows individual roaming clients (road warriors) to connect securely to a central server to access internal network resources or tunnel their internet traffic.

> [!NOTE]
> WireGuard does not differentiate technically between a client and a server; both are simply peers. However, in an RA topology, the "server" maintains a static public IP and open listening port, while "clients" initiate the connection from dynamic IP addresses behind NAT.

## 1. Package Installation and Key Generation

Install the WireGuard packages and generate key pairs for both the central server and the roaming client.

> [!IMPORTANT]
> To simplify administration, client keys are often generated on the server and then securely transferred to the client device. Enforce strict directory permissions (`umask 077`) during generation to protect private keys from unprivileged access.

```Bash
# Install WireGuard on the Debian server
apt install wireguard

# Navigate to the configuration directory and secure file permissions
cd /etc/wireguard
umask 077

# Generate Server Key Pair
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Generate Client Key Pair
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
```

**Command Breakdown & Explanation:**

- `apt install wireguard`: Installs the kernel module tools and `wg-quick` service wrapper.
- `umask 077`: Restricts file creation permissions to `-rw-------` to protect private keys.
- `wg genkey` / `wg pubkey`: Generates base64-encoded Curve25519 cryptographic key pairs for the respective peers.

## 2. IP Packet Forwarding Configuration

For the VPN server to act as a router and forward client traffic to the internet or internal LAN subnets, kernel IP forwarding must be explicitly enabled.

```Bash
# Enable IPv4 packet forwarding in sysctl
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `sysctl -p`: Instructs the kernel to dynamically reload configurations from the `/etc/sysctl.conf` file, activating routing capabilities without requiring a reboot.

## 3. Server Configuration

Create the server configuration file (`wg0.conf`). This file dictates the server's tunnel IP, listening port, NAT (Masquerade) rules for internet access, and registers the client peers.

> [!TIP]
> The `PostUp` and `PostDown` scripts dynamically add and remove `iptables` NAT rules when the tunnel interface starts and stops. Ensure your external interface name matches your system architecture (e.g., `ens18` or `eth0`).

```Bash
# Create and edit the /etc/wireguard/wg0.conf file on the server
cat << 'EOF' > /etc/wireguard/wg0.conf
[Interface]
# Server Private Key (contents of server_private.key)
PrivateKey = <Insert_Server_Private_Key>
Address = 10.255.255.1/24
ListenPort = 51820

# Enable NAT Masquerade so clients can reach the internet/LAN via interface ens18
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE

[Peer]
# Client 1 Public Key (contents of client1_public.key)
PublicKey = <Insert_Client1_Public_Key>
# Lock this specific peer to a single tunnel IP address
AllowedIPs = 10.255.255.2/32
EOF
```

**Command Breakdown & Explanation:**

- `Address = 10.255.255.1/24`: Assigns the VPN gateway IP address to the server's `wg0` interface.
- `ListenPort = 51820`: The UDP port the server will accept incoming connections on.
- `PostUp` / `PostDown`: Executes shell commands to configure stateful routing and Source NAT (Masquerade) for outgoing client traffic.
- `AllowedIPs = 10.255.255.2/32`: Strictly limits Client 1 to utilizing this specific IP address inside the tunnel, preventing IP spoofing between peers.

## 4. Client Configuration

The client configuration block below should be saved to a file (e.g., `client1.conf`) and securely transferred to the user's endpoint device (Windows, macOS, Linux, Android, or iOS).

> [!NOTE]
> Setting `AllowedIPs = 0.0.0.0/0` directs *all* client internet traffic through the VPN (Full Tunnel). To only route traffic destined for a specific corporate LAN, change this to the target subnet (e.g., `AllowedIPs = 10.1.10.0/24` for a Split Tunnel).

```Plaintext
[Interface]
# Client 1 Private Key
PrivateKey = <Insert_Client1_Private_Key>
Address = 10.255.255.2/24
# Optional: Set the DNS server for the client to use while connected
DNS = 1.1.1.1

[Peer]
# Server Public Key
PublicKey = <Insert_Server_Public_Key>
# Server Public WAN IP Address and Listen Port
Endpoint = <Server_Public_IP>:51820
# Route all traffic through the VPN (Full Tunnel)
AllowedIPs = 0.0.0.0/0
# Keep the NAT state alive for roaming clients behind restrictive firewalls
PersistentKeepalive = 25
```

## 5. Service Activation

Activate the WireGuard interface on the server using the `wg-quick` systemd wrapper.

```Bash
# Enable the WireGuard server interface to start at boot and launch it now
systemctl enable wg-quick@wg0 --now
```

**Command Breakdown & Explanation:**

- `wg-quick@wg0`: Reads the configuration, builds the `wg0` interface, applies the IP address, executes the `PostUp` `iptables` rules, and configures the cryptographic kernel routing map.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the server's listening state, active client handshakes, and NAT routing injection on Debian 13 using native diagnostic tools.

### 6.1 Verify WireGuard active listening and peer handshakes

**Command:** `wg show`

**What it checks and variables to look for:**

- **interface**: Must display `wg0` and the configured `listening port` (`51820`).
- **peer**: Must list the Public Key of the registered client.
- **latest handshake**: Will remain blank until the client initiates a connection. Once connected, it must display a recent duration (e.g., `45 seconds ago`).

### 6.2 Verify wg-quick systemd service status

**Command:** `systemctl status wg-quick@wg0`

**What it checks and variables to look for:**

- **Active**: Must be `active (exited)` (expected for oneshot network scripts).
- **Loaded**: Must be `loaded (/lib/systemd/system/wg-quick@.service; enabled)`.

### 6.3 Verify NAT masquerade rule injection

**Command:** `iptables -t nat -L POSTROUTING -v -n`

**What it checks and variables to look for:**

- **Target**: Must display `MASQUERADE`.
- **Out**: Must display your external interface name (e.g., `ens18`).

<!-- Created by: Gergő Téringer, 2026 -->