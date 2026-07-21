<!-- 
---
title: "strongswan-pbased"
author: "Gergő Téringer"
---
 -->
# StrongSwan Policy-Based Site-to-Site VPN

This document provides administrative procedures for configuring a policy-based Site-to-Site (S2S) IPsec Virtual Private Network on Debian 13 (Trixie) using the modern `swanctl` and VICI framework. It demonstrates two distinct authentication opportunities: Pre-Shared Key (PSK) and Certificate-Based (PKI) authentication.

> [!NOTE]
> The `swanctl` utility interacts directly with the `charon` daemon via the VICI plugin, completely replacing the legacy `ipsec.conf` and `stroke` interfaces. StrongSwan remains a policy-based VPN; setting `start_action = trap` automatically installs XFRM kernel policies that intercept and encrypt matching traffic on the fly.

## 1. Package Installation

Install the StrongSwan core daemon along with the `swanctl` management package.

```Bash
# Install the StrongSwan daemon and swanctl utility
apt install strongswan strongswan-swanctl
```

**Command Breakdown & Explanation:**

- `apt install strongswan strongswan-swanctl`: Installs the core IKEv2 `charon` daemon and the modern `swanctl` command-line utility and directory structure.

## 2. IP Packet Forwarding Configuration

To allow the kernel's XFRM framework to route traffic between the encrypted tunnel and the internal LAN interfaces, IP packet forwarding must be explicitly enabled.

```Bash
# Enable IPv4 packet forwarding in sysctl
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

# Apply the kernel parameter changes immediately
sysctl -p
```

**Command Breakdown & Explanation:**

- `sysctl -p`: Instructs the kernel to dynamically reload configurations from the `sysctl.conf` file without requiring a system reboot.

## 3. Segment A: Pre-Shared Key (PSK) Authentication

This segment demonstrates a Site-to-Site configuration relying on a symmetric Pre-Shared Key.

Assume the following topology for this example:

- **Site A Public IP**: `1.1.1.1` | **Site A LAN**: `10.1.0.0/24`
- **Site B Public IP**: `2.2.2.2` | **Site B LAN**: `10.2.0.0/24`

### 3.1 PSK Connection Configuration (swanctl.conf)

```Bash
# Define the PSK connection and secret on Site A
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
    s2s-psk {
        local_addrs  = 1.1.1.1
        remote_addrs = 2.2.2.2
        version = 2
        proposals = aes256-sha256-modp2048
        
        local {
            auth = psk
        }
        remote {
            auth = psk
        }
        
        children {
            s2s-psk-child {
                local_ts  = 10.1.0.0/24
                remote_ts = 10.2.0.0/24
                esp_proposals = aes256gcm16-sha256
                # XFRM Policy Injection: Intercept traffic matching these subnets automatically
                start_action = trap
            }
        }
    }
}

secrets {
    ike-s2s-psk {
        id-a = 1.1.1.1
        id-b = 2.2.2.2
        secret = "SuperSecretKey123"
    }
}
EOF
```

**Command Breakdown & Explanation:**

- `version = 2`: Strictly forces the IKEv2 protocol.
- `local_ts` / `remote_ts`: Defines the Traffic Selectors (subnets) that will trigger the policy.
- `start_action = trap`: Installs the routing policies into the kernel immediately upon configuration load. The actual VPN connection (Phase 1/2) is only established dynamically when a packet matches the traffic selectors.
- `secrets { ike-s2s-psk ... }`: Binds the Pre-Shared Key to the specific peer IP addresses.

## 4. Segment B: Certificate-Based (PKI) Authentication

This segment demonstrates replacing the PSK with a Public Key Infrastructure (X.509 certificates). PKI ensures distinct cryptographic identities and simplifies scaling.

> [!IMPORTANT]
> The `swanctl` utility strictly relies on standard directory paths for cryptography. Certificates and keys must be placed in their exact designated folders under `/etc/swanctl/`.

### 4.1 Certificate Placement

```Bash
# Copy the PKI files to the required swanctl directories on Site A
cp /ca/ca.crt /etc/swanctl/x509ca/
cp /ca/site-a.crt /etc/swanctl/x509/
cp /ca/site-a.key /etc/swanctl/private/

# Secure the private key directory
chmod 700 /etc/swanctl/private
chmod 600 /etc/swanctl/private/site-a.key
```

## 4.2 PKI Connection Configuration (swanctl.conf)

```Bash
# Overwrite the swanctl.conf file on Site A for PKI utilization
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
    s2s-cert {
        local_addrs  = 1.1.1.1
        remote_addrs = 2.2.2.2
        version = 2
        proposals = aes256-sha256-modp2048
        
        local {
            auth = pubkey
            certs = site-a.crt
            id = "C=HU, O=Ceg, CN=site-a.domain.name"
        }
        remote {
            auth = pubkey
            id = "C=HU, O=Ceg, CN=site-b.domain.name"
        }
        
        children {
            s2s-cert-child {
                local_ts  = 10.1.0.0/24
                remote_ts = 10.2.0.0/24
                esp_proposals = aes256gcm16-sha256
                start_action = trap
            }
        }
    }
}
EOF
```

**Command Breakdown & Explanation:**

- `auth = pubkey`: Instructs the daemon to utilize X.509 certificates rather than a shared secret.
- `certs = site-a.crt`: Tells `swanctl` to load this specific file from the `/etc/swanctl/x509/` directory to authenticate itself to the remote peer. The matching private key is automatically sourced from the `private` folder.
- `id`: Defines the expected X.509 Subject Distinguished Names (DN). `swanctl` will verify that the Subject presented in Site B's certificate exactly matches the `remote` ID string.

## 5. Service Activation and Loading

Once the configurations are established and certificates are placed, you must start the daemon and load the `swanctl` parameters.

```Bash
# Enable the swanctl service to start on boot
systemctl enable strongswan-swanctl --now

# Manually trigger a reload of all swanctl configurations, connections, and credentials
swanctl --load-all
```

**Command Breakdown & Explanation:**

- `swanctl --load-all`: Connects to the `charon` daemon via the VICI interface to parse `/etc/swanctl/swanctl.conf`, loads all certificates/keys from the directory structure, and injects the `trap` XFRM policies into the Linux kernel's routing tables.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the configuration parsing, active Security Associations (SAs), and kernel policy injections on Debian 13 using standard `swanctl` diagnostic commands.

### 6.1 Verify loaded configurations and kernel policies

**Command:** `swanctl --list-conns`

**What it checks and variables to look for:**

- **Connection Name**: Must display your configured connection (e.g., `s2s-cert`).
- **Child SAs**: Must display `s2s-cert-child: TUNNEL` and list the `local` and `remote` subnets.
- **State**: Look for the `trap` keyword, confirming the kernel is actively monitoring for traffic to encrypt.

### 6.2 Verify active IPsec Security Associations (SAs)

**Command:** `swanctl --list-sas`

**What it checks and variables to look for:**

- **IKE_SA Status**: Once traffic triggers the connection, this must display `ESTABLISHED` indicating a successful Phase 1 handshake.
- **CHILD_SA Status**: Must display `INSTALLED` accompanied by the matching subnet pair (e.g., `10.1.0.0/24 === 10.2.0.0/24`), confirming Phase 2 ESP encryption is actively securing packets.

### 6.3 Verify end-to-end tunnel connectivity

**Command:** `ping -c 4 10.2.0.10` *(executed from a host on Site A LAN)*

**What it checks and variables to look for:**

- **Packet Loss**: Must show `0% packet loss`. Because this is a policy-based VPN, you generally cannot ping from the gateway's external IP to the remote subnet; the traffic must originate from an IP matching the defined `local_ts` to trigger the XFRM encryption policy natively.

<!-- Created by: Gergő Téringer, 2026 -->