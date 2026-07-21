<!-- 
---
title: "DHCP Relay"
author: "Gergő Téringer"
---
 -->
# DHCP Relay

This document provides administrative procedures for configuring DHCP Relay agents on Debian 13 (Trixie). It covers both the traditional `isc-dhcp-relay` daemon for standard broadcast networks and the lightweight `dhcp-helper` daemon, which establishes Layer 3 connections suitable for passing DHCP traffic across VPN tunnels or unsupported interface types.

## 1. Traditional ISC-DHCP-Relay Configuration

The `isc-dhcp-relay` package listens for DHCP broadcast requests from clients on one subnet and forwards them as unicast packets to a DHCP server on a different subnet.

> [!NOTE]
> If installing in an offline environment without internet access, ensure the third Debian installation ISO is attached to the system repository sources before executing the installation command.

```bash
# Install the ISC DHCP Relay package
apt install isc-dhcp-relay
```

**Command Breakdown & Explanation:**

- `apt install isc-dhcp-relay`: Installs the traditional ISC DHCP relay daemon. During installation, `dpkg` will launch an interactive prompt requesting the following configuration details in order:

  1. **Servers:** The IP address(es) of the upstream DHCP server(s) to forward requests to.
  2. **Interfaces:** A space-separated list of network interfaces the relay should listen on and forward from (e.g., `eth0 eth1`). You must include both the client-facing interface and the server-facing interface.
  3. **Additional options:** Extra command-line arguments (usually left blank).

### 1.1 Post-Installation Configuration

If you need to modify the server targets or interfaces after the initial installation wizard, you must edit the daemon's environment defaults file.

```bash
# Edit the ISC DHCP Relay defaults file
nano /etc/default/isc-dhcp-relay

# Restart the service to apply changes
systemctl restart isc-dhcp-relay
```

**Command Breakdown & Explanation:**

- `/etc/default/isc-dhcp-relay`: The configuration file where `SERVERS`, `INTERFACES`, and `OPTIONS` variables are stored based on your initial installation prompts.
- `systemctl restart isc-dhcp-relay`: Binds the daemon to the newly specified interfaces and applies the updated upstream server IP addresses.

## 2. DHCP-Helper for Tunnel Interfaces

Standard `isc-dhcp-relay` relies on raw Layer 2 packet sockets, which often fail or are unsupported across virtual tunnel interfaces (like GRE, OpenVPN, or IPsec VTI). The `dhcp-helper` package is the recommended alternative, as it utilizes standard Layer 3 UDP connections to proxy DHCP traffic to upstream servers.

```bash
# Install the DHCP Helper package
apt install dhcp-helper

# Edit the DHCP Helper options
tee /etc/default/dhcp-helper > /dev/null << 'EOF'
# Example configuration targeting two DHCP servers and listening on ens224
DHCPHELPER_OPTS="-s 172.20.2.1 -s 172.20.2.2 -i ens224"
EOF

# Restart the service to apply changes
systemctl restart dhcp-helper
```

**Command Breakdown & Explanation:**

- `apt install dhcp-helper`: Installs the lightweight DHCP relay alternative designed for complex routing and tunnel topologies.
- `DHCPHELPER_OPTS`: Defines the runtime arguments passed to the daemon upon startup.
- `-s 172.20.2.1`: Specifies the upstream DHCP server IP address. Multiple `-s` flags can be used for redundancy.
- `-i ens224`: Explicitly defines the interface to listen on for client broadcast requests.
- `systemctl restart dhcp-helper`: Applies the new options and initiates the listener process.

## 3. Verification and Troubleshooting

> [!NOTE]
> Validate the relay daemon status and network port bindings on Debian 13 using standard administrative commands. Ensure only one DHCP relay daemon (`isc-dhcp-relay` or `dhcp-helper`) is running simultaneously to prevent UDP port conflicts.

### 3.1 Verify isc-dhcp-relay service status

**Command:** `systemctl status isc-dhcp-relay`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/isc-dhcp-relay.service; enabled)`

### 3.2 Verify dhcp-helper service status

**Command:** `systemctl status dhcp-helper`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/dhcp-helper.service; enabled)`

### 3.3 Verify network listening state on DHCP ports

**Command:** `ss -tulnp | grep :67`

**What it checks and variables to look for:**

- **State**: Must be `UNCONN`
- **Local Address:Port**: Must display `*:67` or `0.0.0.0:67`
- **Process**: Must display `"dhcrelay"` or `"dhcp-helper"` mapping to the active PID

<!-- Created by: Gergő Téringer, 2026 -->