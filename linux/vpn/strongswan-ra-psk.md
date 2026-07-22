<!-- 
---
title: "StrongSwan Remote Access VPN (PSK)"
author: "Gergő Téringer"
---
 -->
# StrongSwan Remote Access VPN (PSK)

This document provides administrative procedures for configuring a Remote Access (RA) IPsec Virtual Private Network using StrongSwan with Pre-Shared Key (PSK) authentication on Debian 13 (Trixie). It utilizes the modern `swanctl` utility and assigns virtual IP (vIP) addresses to roaming clients from a central pool.

> [!NOTE]
> This topology consists of an internal server LAN (`10.1.1.0/24`), the StrongSwan VPN gateway (`RA-RTR` at `195.199.203.100`), and a roaming client (`RA-CLT` at `195.199.203.97`). PSK provides a simpler deployment mechanism compared to EAP-TLS by using a symmetric secret key shared between the gateway and the client.

## 1. Server Gateway Installation and Configuration

Install the modern StrongSwan components. Because we are using PSK instead of EAP-TLS, the extra plugins package is not strictly required for authentication.

```Bash
# Install the core charon daemon and swanctl
apt install charon-systemd strongswan-swanctl

# Define the connection parameters, address pool, and PSK secret on the RA-RTR server
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
  remoteaccess {
    version = 2
    
    local_addrs = 195.199.203.100
    pools = ra_pool

    local {
      auth = psk
      id = RA-RTR.kontozo.hu
    }
    remote {
      auth = psk
      id = remoteaccess@kontozo.hu
    }
    children {
      net {
        local_ts = 10.1.1.0/24
      }
    }
  }
}

pools {
  ra_pool {
    addrs = 10.1.200.0/24
    dns = 10.1.1.1
  }
}

secrets {
  ike-ra-psk {
    id-1 = RA-RTR.kontozo.hu
    id-2 = remoteaccess@kontozo.hu
    secret = "SuperSecretPsk123!"
  }
}

# Include config snippets
include conf.d/*.conf
EOF
```

**Command Breakdown & Explanation:**

- `local { auth = psk }`: Instructs the server to authenticate itself using a Pre-Shared Key.
- `id = ...`: Defines an explicit identification string. Since PSKs do not have embedded Subject Alternative Names like certificates, explicit IDs help the `charon` daemon select the correct secret from the `secrets` block.
- `secrets { ike-ra-psk { ... } }`: Binds the symmetric secret to the specific local and remote IDs.

## 2. Roaming Client Configuration

The roaming client requires the exact same package dependencies. It must be configured to request an IP address, authenticate via PSK using the matching ID strings, and utilize the shared secret.

```Bash
# Install the core charon daemon and swanctl on the client
apt install charon-systemd strongswan-swanctl

# Define the connection parameters and PSK secret on the RA-CLT client
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
  remoteaccess {
    version = 2
    
    remote_addrs = 195.199.203.100
    vips = 0.0.0.0
    dpd_delay = 3s

    local {
      auth = psk
      id = remoteaccess@kontozo.hu
    }
    remote {
      auth = psk
      id = RA-RTR.kontozo.hu
    }
    children {
      net {
        remote_ts = 10.1.1.0/24
        start_action = trap|start
        dpd_action = clear
      }
    }
  }
}

secrets {
  ike-ra-psk {
    id-1 = remoteaccess@kontozo.hu
    id-2 = RA-RTR.kontozo.hu
    secret = "SuperSecretPsk123!"
  }
}

# Include config snippets
include conf.d/*.conf
EOF
```

Next, adjust the core StrongSwan daemon settings on the client to detect dead peer connections (broken tunnels) faster.

```Bash
# Append custom retransmission settings to /etc/strongswan.conf
cat << 'EOF' >> /etc/strongswan.conf
charon {
  retransmit_base = 1.1
  retransmit_tries = 3
  retransmit_timeout = 2
}
EOF
```

**Command Breakdown & Explanation:**

- `vips = 0.0.0.0`: Requests a Virtual IP address from the server's `ra_pool`.
- `remote { auth = psk }`: Enforces that the server must also authenticate using the matching PSK.
- `start_action = trap|start`: Installs the XFRM routing policies immediately and initiates the tunnel connection to the server.

## 3. Service Activation and Loading

Once the configurations are established and the secrets are identical on both ends, you must start the daemon and load the `swanctl` parameters.

```Bash
# Enable the swanctl services on both the Server and Client
systemctl enable strongswan-swanctl --now

# Reload the swanctl configurations to immediately apply the connection and secrets
swanctl --load-all
```

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the assigned vIP, the active Security Associations (SAs), and end-to-end routing on Debian 13 using standard diagnostic tools.

### 4.1 Verify live daemon logs

**Command:** `journalctl -fxeu strongswan` *(Run on either Server or Client)*

**What it checks and variables to look for:**

- **Authentication**: Must display `IKE_SA remoteaccess[1] established` confirming the PSK matched.
- **Virtual IP**: Must show `assigning virtual IP 10.1.200.1 to peer` (on the server) or `installing new virtual IP 10.1.200.1` (on the client).

### 4.2 Verify active IPsec Security Associations

**Command:** `swanctl --list-sas` *(Run on either Server or Client)*

**What it checks and variables to look for:**

- **IKE_SA Status**: Must display `ESTABLISHED` indicating the PSK Phase 1 handshake succeeded.
- **CHILD_SA Status**: Must display `INSTALLED` and confirm the internal routing mapping (e.g., `10.1.200.1/32 === 10.1.1.0/24`).

### 4.3 Verify end-to-end tunnel connectivity

**Command:** `ping -c 4 10.1.1.1` *(Run from the Client)*

**What it checks and variables to look for:**

- **Packet Loss**: Must show `0% packet loss`. Packets originating from the client's assigned vIP (`10.1.200.x`) will successfully route through the encrypted tunnel to the internal LAN network.

<!-- Created by: Gergő Téringer, 2026 -->