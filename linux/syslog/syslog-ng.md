<!-- 
---
title: "Syslog-NG"
author: "Gergő Téringer"
---
 -->
# Syslog-NG

This document provides administrative procedures for installing and configuring the `syslog-ng` logging daemon on Debian 13 (Trixie). It covers core component syntax (sources, filters, destinations), network transport protocols (IETF vs. BSD), and setting up secure centralized log collection over TLS.

> [!NOTE]
> The primary configuration file is `/etc/syslog-ng/syslog-ng.conf`. For modularity and clean separation, all custom configurations should be placed inside the `/etc/syslog-ng/conf.d/` directory with a `.conf` extension, as the main configuration file automatically includes them.

## 1. Package Installation and Service Management

The `syslog-ng` package replaces standard logging utilities (like `rsyslog`) to provide advanced routing, filtering, and network transport capabilities.

```Bash
# Install the syslog-ng daemon package
apt install syslog-ng

# Enable the service to start at boot and launch it immediately
systemctl enable syslog-ng@default --now
```

**Command Breakdown & Explanation:**

- `apt install syslog-ng`: Installs the core logging daemon and default local configuration.
- `systemctl enable ...`: Activates the service instance.

## 2. Core Configuration Components

A `syslog-ng` configuration is built using four primary building blocks. The default source that collects local system logs is typically predefined as `s_src`.

### 2.1 Sources

Defines where the daemon collects log messages (e.g., local system sockets, files, or network ports).

```Plaintext
source s_name {
    system();
};
```

### 2.2 Filters

Filters allow you to route specific logs to different destinations based on criteria like log level, facility, or the generating program.

```Plaintext
filter f_name {
    level(info) and facility(mail) and not program("dhcpd");
};
```

### 2.3 Destinations

Defines where the processed logs are sent, such as local files, network endpoints, or specific TTY terminals.

> [!TIP]
> If you want to push specific emergency logs directly to an active terminal screen, you can define a destination like `file("/dev/tty5");`.

```Plaintext
destination d_name {
    file("/var/log/custom.log");
};
```

### 2.4 Log Statements

The `log` statement is the active rule that binds sources, optional filters, and destinations together to form a complete logging pipeline.

```Plaintext
log {
    source(s_name);
    filter(f_name); # Optional
    destination(d_name);
};
```

## 3. Network Transport and Peer Verification

`syslog-ng` supports two distinct protocols for network transport. The protocol keyword must be defined within a `source` (for servers receiving logs) or a `destination` (for clients sending logs).

- **IETF Syslog Protocol (`syslog` keyword)**: The modern standard for syslog.
- **BSD Syslog Protocol (`network` keyword)**: The legacy protocol standard.

### 3.1 Syntax for Network Transport

```Plaintext
# For modern IETF protocol:
syslog {
    ip-protocol(4) # Set to 6 to listen on both IPv6 and IPv4
    port(6514)     # Define a listening/sending port between 1-65536
    transport("tls") # Options: udp, tcp, tls
    # TLS block follows...
};

# For legacy BSD protocol:
network {
    ip-protocol(4)
    port(6514)
    transport("tls")
    # TLS block follows...
};
```

### 3.2 TLS Peer Verification Options

When utilizing `transport("tls")`, the `peer-verify()` option dictates how strictly the daemon validates client or server certificates. The default value is `required-trusted`.

| Option | No Certificate Provided | Invalid Certificate | Valid Certificate |
| :--- | :---: | :---: | :---: |
| **optional-untrusted** | TLS-encryption | TLS-encryption | TLS-encryption |
| **optional-trusted** | TLS-encryption | Rejected connection | TLS-encryption |
| **require-untrusted** | Rejected connection | TLS-encryption | TLS-encryption |
| **require-trusted** | Rejected connection | Rejected connection | TLS-encryption |

## 4. Encrypted TLS Transport Configuration

To securely transfer logs between endpoints, you must utilize X.509 certificates. Ensure that the CA certificate used to sign the endpoint keys is trusted by both the server and the client.

> [!IMPORTANT]
> The paths to the CA certificate (`ca-file`), endpoint certificate (`cert-file`), and endpoint private key (`key-file`) must be readable by the `syslog-ng` daemon.

### 4.1 Server-Side Configuration  

Place this configuration in `/etc/syslog-ng/conf.d/server-tls.conf` on the central logging server. It defines a network source listening securely on port 6514 and routes incoming DHCP logs to a dedicated file.

```Plaintext
source s_dhcp {
    syslog(
        ip-protocol(4)
        port(6514)
        transport("tls")
        tls (
            cert-file("/ca/SRV.pem")
            key-file("/ca/SRV.key")
            ca-file("/ca/CA.crt")
            ca-dir("/ca/")
        )
    );
};

destination d_dhcp {
    file("/var/log/dhcp_remote.log");
};

log {
    source(s_dhcp);
    destination(d_dhcp);
};
```

### 4.2 Client-Side Configuration

Place this configuration in `/etc/syslog-ng/conf.d/client-tls.conf` on the client machine. It defines a secure network destination, filters local DHCP traffic, and pushes those logs to the central server.

```Plaintext
desination d_remote_dhcp {
    syslog(
        "SRV.lego.dk"
        port(6514)
        transport("tls")
        tls(
            cert-file("/ca/CLT.pem")
            key-file("/ca/CLT.key")
            ca-file("/ca/CA.crt")
        )
    );
};

filter f_dhcp {
    program("dhcpd") or program("dhclient");
};

log {
    source(s_src);
    filter(f_dhcp);
    destination(d_remote_dhcp);
};

# Example to send everything EXCEPT the filtered DHCP traffic:
# log {
#     source(s_src);
#     filter { not filter(f_dhcp); };
#     destination(d_remote_dhcp);
# };
```

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the `syslog-ng` service status, configuration syntax, and active listening ports on Debian 13 using standard administrative tools.

### 5.1 Verify syslog-ng configuration syntax

**Command:** `syslog-ng -s`

**What it checks and variables to look for:**

- **Syntax output**: Must complete silently. If any syntax or structural errors exist in the `.conf` files, it will explicitly output the exact line number and failing block.

### 5.2 Verify syslog-ng service status

**Command:** `systemctl status syslog-ng@default`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/syslog-ng@.service; enabled)`

### 5.3 Verify network listening state on central server (Port 6514)

**Command:** `ss -tulnp | grep 6514`

**What it checks and variables to look for:**

- **State**: Must display `LISTEN`
- **Process**: Must display `syslog-ng` bound to `*:6514` or `0.0.0.0:6514`

<!-- Created by: Gergő Téringer, 2026 -->