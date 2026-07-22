<!-- 
---
title: "StrongSwan Remote Access VPN (Certificate)"
author: "Gergő Téringer"
---
 -->
# StrongSwan Remote Access VPN (Certificate)

This document provides administrative procedures for configuring a Remote Access (RA) IPsec Virtual Private Network using StrongSwan with standard Public Key (Certificate) authentication on Debian 13 (Trixie). It utilizes the modern `swanctl` utility and assigns virtual IP (vIP) addresses to roaming clients from a central pool.

> [!NOTE]
> This topology utilizes the standard X.509 `pubkey` authentication method rather than EAP-TLS or PSK. Both the gateway (`RA-RTR` at `195.199.203.100`) and the roaming client (`RA-CLT` at `195.199.203.97`) mutually authenticate each other directly using their respective certificates signed by a trusted Certificate Authority.

## 1. Server Gateway Installation and Certificate Placement

Install the modern StrongSwan components. Because we are using native `pubkey` authentication, the core daemon natively supports reading the X.509 certificates without requiring additional EAP plugins.

```Bash
# Install the core charon daemon and swanctl utility
apt install charon-systemd strongswan-swanctl

# Copy the PKI files to the required swanctl directories
cp CAcert.pem /etc/swanctl/x509ca/
cp serverCert.pem /etc/swanctl/x509/
cp serverKey.pem /etc/swanctl/private/

# Secure the private key directory
chmod 700 /etc/swanctl/private
chmod 600 /etc/swanctl/private/serverKey.pem
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `apt install charon-systemd strongswan-swanctl`: Installs the systemd-integrated IKE daemon and the modern VICI configuration parser.
- `cp ... /etc/swanctl/...`: Places the Root CA, server certificate, and private key into the designated folders where `swanctl` naturally searches for cryptographic credentials.

## 2. Server Gateway Configuration (swanctl.conf)

Configure the server to listen for incoming connections, enforce `pubkey` mutual authentication, and distribute Virtual IPs (vIPs) alongside DNS parameters from a designated pool.

```Bash
# Define the connection parameters and address pool on the RA-RTR server
cat << 'EOF' > /etc/swanctl/swanctl.conf
connections {
  remoteaccess {
    version = 2
    
    local_addrs = 195.199.203.100
    pools = ra_pool

    local {
      auth = pubkey
      certs = serverCert.pem
      id = "C=HU, O=Kontozo, CN=RA-RTR.kontozo.hu"
    }
    remote {
      auth = pubkey
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
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `local { auth = pubkey }`: Dictates that the server will authenticate itself using standard X.509 RSA/ECDSA certificate signatures.
- `certs = serverCert.pem`: Tells the daemon to load this specific certificate from `/etc/swanctl/x509/`. The matching private key in the `private` folder will be linked automatically.
- `remote { auth = pubkey }`: Enforces that connecting clients must also provide a valid certificate signed by the Root CA located in `/etc/swanctl/x509ca/`.

## 3. Roaming Client Installation and Certificate Placement

The roaming client requires the exact same package dependencies and directory structure. Ensure the client's unique certificate and private key are securely transferred to the device.

```Bash
# Install the core charon daemon and swanctl on the client
apt install charon-systemd strongswan-swanctl

# Copy the PKI files to the required swanctl directories
cp CAcert.pem /etc/swanctl/x509ca/
cp clientCert.pem /etc/swanctl/x509/
cp clientKey.pem /etc/swanctl/private/

# Secure the private key directory
chmod 700 /etc/swanctl/private
chmod 600 /etc/swanctl/private/clientKey.pem
```

## 4. Roaming Client Configuration

The client must be configured to request an IP address, authenticate via its local `pubkey`, and verify the server's identity string.

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
      auth = pubkey
      certs = clientCert.pem
      id = "C=HU, O=Kontozo, CN=remoteaccess@kontozo.hu"
    }
    remote {
      auth = pubkey
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
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `vips = 0.0.0.0`: Requests a Virtual IP address dynamically from the server's pool.
- `remote { id = "..." }`: Enforces strict identity checking. The server's certificate Subject DN must exactly match this string to prevent rogue gateway attacks.
- `start_action = trap|start`: Installs the XFRM routing policies immediately and initiates the tunnel connection to the server.

## 5. Service Activation and Loading

Once the configurations are established and certificates are placed on both ends, you must start the daemon and load the `swanctl` parameters.

```Bash
# Enable the swanctl services on both the Server and Client
systemctl enable strongswan-swanctl --now

# Reload the swanctl configurations to immediately parse the certificates and connections
swanctl --load-all
```

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate the assigned vIP, the parsed X.509 certificates, and the active Security Associations (SAs) on Debian 13 using standard diagnostic tools.

### 6.1 Verify loaded certificates

**Command:** `swanctl --list-certs` *(Run on either Server or Client)*

**What it checks and variables to look for:**

Before lists place a blank line!

- **X509 Certificates**: Must display the subject of the locally loaded `.pem` file and confirm it has successfully located the corresponding private key (`has private key`).

### 6.2 Verify active IPsec Security Associations

**Command:** `swanctl --list-sas` *(Run on either Server or Client)*

**What it checks and variables to look for:**

Before lists place a blank line!

- **IKE_SA Status**: Must display `ESTABLISHED` indicating the `pubkey` Phase 1 mutual authentication succeeded.
- **CHILD_SA Status**: Must display `INSTALLED` and confirm the internal routing mapping (e.g., `10.1.200.1/32 === 10.1.1.0/24`).

### 6.3 Verify end-to-end tunnel connectivity

**Command:** `ping -c 4 10.1.1.1` *(Run from the Client)*

**What it checks and variables to look for:**

Before lists place a blank line!

- **Packet Loss**: Must show `0% packet loss`. Packets originating from the client's assigned vIP (`10.1.200.x`) will successfully route through the encrypted tunnel to the internal LAN network.

<!-- Created by: Gergő Téringer, 2026 -->