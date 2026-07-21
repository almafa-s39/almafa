<!-- 
---
title: "strongswan-ra-eap-tls"
author: "Gergő Téringer"
---
 -->
# Strongswan Remote Access VPN (EAP-TLS)

This document provides administrative procedures for configuring a Remote Access (RA) IPsec Virtual Private Network using StrongSwan with EAP-TLS authentication on Debian 13 (Trixie). It utilizes the modern `swanctl` utility and assigns virtual IP (vIP) addresses to roaming clients from a central pool.

> [!NOTE]
> The topology consists of an internal server LAN (`10.1.1.0/24`), the StrongSwan VPN gateway (`RA-RTR` at `195.199.203.100`), and a roaming client (`RA-CLT` at `195.199.203.97`). EAP-TLS provides robust, certificate-based mutual authentication.

## 1. Server Gateway Installation and Certificate Placement

Install the modern StrongSwan components and the extra plugins required for EAP-TLS authentication. Once installed, copy the pre-generated CA, server certificate, and private key to the strict `swanctl` directory structure.

> [!IMPORTANT]
> The server certificate must contain a Subject Alternative Name (SAN) matching its public IP address (e.g., `IP=195.199.203.100`).

```Bash
# Install the core charon daemon, swanctl, and EAP plugins
apt install charon-systemd strongswan-swanctl libcharon-extra-plugins

# Copy the PKI files to the required swanctl directories
cp CAcert.pem /etc/swanctl/x509ca/
cp serverCert.pem /etc/swanctl/x509/
cp serverKey.pem /etc/swanctl/private/

# Secure the private key directory
chmod 700 /etc/swanctl/private
chmod 600 /etc/swanctl/private/serverKey.pem
```

**Command Breakdown & Explanation:**

- `apt install charon-systemd ...`: Installs the systemd-integrated IKE daemon and plugins required for EAP (Extensible Authentication Protocol).
- `cp ... /etc/swanctl/...`: Places certificates and keys into the designated folders where the VICI plugin natively searches for credentials.

## 2. Server Gateway Configuration (swanctl.conf)

Configure the server to listen for incoming connections, enforce EAP-TLS, and distribute Virtual IPs (vIPs) alongside DNS parameters from a designated pool.

```Bash
# Define the connection parameters and address pool on the RA-RTR server
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
  remoteaccess {
    version = 2
    
    local_addrs = 195.199.203.100
    pools = ra_pool

    local {
      auth = eap-tls
      certs = serverCert.pem
    }
    remote {
      auth = eap-tls
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

# Include config snippets
include conf.d/*.conf
EOF

# Restart the StrongSwan service to load the new configuration
systemctl restart strongswan
```

**Command Breakdown & Explanation:**

- `pools = ra_pool`: Links the connection to the IP address pool defined at the bottom of the configuration.
- `local { auth = eap-tls }`: Dictates that the server will authenticate itself using its certificate inside an EAP payload.
- `remote { auth = eap-tls }`: Enforces that connecting clients must also provide a valid certificate via EAP-TLS.
- `local_ts = 10.1.1.0/24`: Instructs connecting clients that the server routes traffic for this internal LAN subnet.

## 3. Roaming Client Installation and Certificate Placement

The roaming client requires the exact same package dependencies as the server. It also needs its own specific certificate containing an email or identifying SAN.

> [!IMPORTANT]
> The client certificate must contain a Subject Alternative Name (SAN) identifying the user (e.g., `email=remoteaccess@kontozo.hu`).

```Bash
# Install the core charon daemon, swanctl, and EAP plugins on the client
apt install charon-systemd strongswan-swanctl libcharon-extra-plugins

# Copy the PKI files to the required swanctl directories
cp CAcert.pem /etc/swanctl/x509ca/
cp clientCert.pem /etc/swanctl/x509/
cp clientKey.pem /etc/swanctl/private/

# Secure the private key directory
chmod 700 /etc/swanctl/private
chmod 600 /etc/swanctl/private/clientKey.pem
```

## 4. Roaming Client Configuration

The client must be configured to request an IP address, authenticate via EAP-TLS, and adjust its retransmission parameters to quickly tear down broken tunnels.

```Bash
# Define the connection parameters on the RA-CLT client
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
  remoteaccess {
    version = 2
    
    remote_addrs = 195.199.203.100
    vips = 0.0.0.0
    dpd_delay = 3s

    local {
      auth = eap-tls
      certs = clientCert.pem
    }
    remote {
      auth = eap-tls
      id = "C=HU, O=Kontozo, CN=RA-RTR.kontozo.hu"
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

# Restart the StrongSwan service to initiate the connection
systemctl restart strongswan
```

**Command Breakdown & Explanation:**

- `vips = 0.0.0.0`: Requests a Virtual IP address from the server's `ra_pool`.
- `remote { id = "..." }`: Enforces strict identity checking. The server's certificate Subject DN must exactly match this string.
- `start_action = trap|start`: Installs the XFRM policies immediately and attempts to initiate the tunnel connection dynamically.
- `dpd_action = clear`: Instructs the daemon to tear down the connection and remove routing/DNS rules if the Dead Peer Detection (DPD) timeout is reached.
- `retransmit_...`: Modifies the IKE exchange timeouts to ensure the interface fails cleanly if network connectivity drops.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the assigned vIP, the active Security Associations (SAs), and end-to-end routing on Debian 13 using standard diagnostic tools.

### 5.1 Verify live daemon logs

**Command:** `journalctl -fxeu strongswan` *(Run on either Server or Client)*

**What it checks and variables to look for:**

- **Authentication**: Must display `EAP-TLS authentication of '...' successful`.
- **Virtual IP**: Must show `assigning virtual IP 10.1.200.1 to peer` (on the server) or `installing new virtual IP 10.1.200.1` (on the client).

### 5.2 Verify active IPsec Security Associations

**Command:** `swanctl --list-sas` *(Run on either Server or Client)*

**What it checks and variables to look for:**

- **IKE_SA Status**: Must display `ESTABLISHED` indicating the EAP-TLS Phase 1 handshake succeeded.
- **CHILD_SA Status**: Must display `INSTALLED` and confirm the internal routing mapping (e.g., `10.1.200.1/32 === 10.1.1.0/24`).

### 5.3 Verify end-to-end tunnel connectivity

**Command:** `ping -c 4 10.1.1.1` *(Run from the Client)*

**What it checks and variables to look for:**

- **Packet Loss**: Must show `0% packet loss`. Packets originating from the client's assigned vIP (`10.1.200.x`) will successfully route through the encrypted tunnel to the internal LAN network.

<!-- Created by: Gergő Téringer, 2026 -->