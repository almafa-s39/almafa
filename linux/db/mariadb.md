<!-- 
---
title: "MariaDB Server Installation and Baseline Configuration Guide"
author: "Gergő Téringer"
---
 -->
# MariaDB Server Installation and Baseline Configuration Guide

This document provides administrative procedures for deploying, securing, and configuring MariaDB Server on Debian 13 (Trixie). It covers package installation, automated or interactive security hardening, user and database provisioning, network listener binding, and diagnostic verification steps.

> [!NOTE]
> On Debian 13, MariaDB utilizes `unix_socket` authentication by default for the local `root` account, permitting administrative access via `mariadb` without requiring a plain-text password prompt during local terminal sessions.

## 1. Package Installation and Service Management

> [!IMPORTANT]
> Refresh local package index files prior to installation to ensure binaries are retrieved from the latest Debian 13 repository snapshots.

```bash
# Update Debian package repositories and install MariaDB Server and Client
apt install mariadb-server mariadb-client

# Enable and start the MariaDB daemon
systemctl enable mariadb --now
```

**Command Breakdown & Explanation:**

- `apt install mariadb-server mariadb-client`: Installs the MariaDB relational database server engine along with the command-line client interface tool (`mariadb`).
- `systemctl enable mariadb --now`: Configures the MariaDB systemd service unit to launch automatically during system boot and immediately starts the service.

## 2. Database Security Hardening

Default database installations include sample anonymous user accounts, test databases, and unrestricted remote administrative access privileges. Running the secure installation tool eliminates these security vectors.

> [!WARNING]
> Disabling remote root login enforces administrative connections strictly over encrypted SSH sessions or local socket channels, eliminating automated brute-force attacks against the root database identity.

```bash
# Execute the interactive MariaDB security hardening wizard
mariadb-secure-installation
```

**Command Breakdown & Explanation:**

- `mariadb-secure-installation`: Launches an interactive script to set/verify unix_socket or root credentials, remove anonymous user accounts, disallow remote root logins, drop the default `test` database, and reload privilege grant tables.

## 3. Database and User Provisioning

Production applications must utilize dedicated databases and restricted non-root user accounts adhering to the Principle of Least Privilege.

> [!TIP]
> Specify explicit network constraints (e.g., `'app_user'@'10.10.20.%'`) when creating accounts to restrict database access exclusively to authorized application subnet ranges.

```sql
-- Create a dedicated application database with UTF-8 encoding
CREATE DATABASE vpn_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Provision a restricted application user bound to a specific network range
CREATE USER 'app_user'@'10.10.20.%' IDENTIFIED BY 'SecureAppPassword123!';

-- Grant privileges limited strictly to the application database
GRANT ALL PRIVILEGES ON vpn_db.* TO 'app_user'@'10.10.20.%';

-- Reload privilege tables to activate changes immediately
FLUSH PRIVILEGES;
```

**Command Breakdown & Explanation:**

- `CREATE DATABASE vpn_db...`: Provisions a database using 4-byte UTF-8 encoding (`utf8mb4`) for standard international character set support.
- `CREATE USER 'app_user'@'10.10.20.%'...`: Configures a database identity restricted to connections originating from the `10.10.20.0/24` network segment.
- `GRANT ALL PRIVILEGES ON vpn_db.*...`: Assigns full database manipulation rights solely to objects located inside `vpn_db`.
- `FLUSH PRIVILEGES`: Forces the server engine to reload grant table structures into active memory.

## 4. Network Listener and Remote Access Configuration

By default, MariaDB listens exclusively on the loopback adapter (`127.0.0.1`). To permit remote network connections from external application servers, modify the bind address setting in the configuration directory.

> [!CAUTION]
> Binding MariaDB to all interfaces (`0.0.0.0`) exposes the service to attached network interfaces. Host-level firewall rules (such as `nftables` or `ufw`) must be enforced to restrict TCP port 3306 traffic strictly to trusted source IPs.

```bash
# Update the bind-address setting in 50-server.cnf to listen on all IPv4 addresses
sed -i 's/^bind-address\s*=\s*127.0.0.1/bind-address = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf

# Restart the MariaDB service to apply network configuration changes
systemctl restart mariadb
```

**Command Breakdown & Explanation:**

- `sed -i 's/.../...' /etc/mysql/mariadb.conf.d/50-server.cnf`: Modifies the `bind-address` configuration parameter to `0.0.0.0`, enabling TCP connections across all active network interfaces.
- `systemctl restart mariadb`: Restarts the daemon process to rebind the TCP listening socket to port 3306.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate MariaDB service status, listening socket state, user access privileges, and remote database connectivity on Debian 13 using standard administrative commands.

### 5.1 Verify MariaDB service status

**Command:** `systemctl status mariadb`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/mariadb.service; enabled)`

### 5.2 Verify network listening state on TCP port 3306

**Command:** `ss -tuln | grep 3306`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `0.0.0.0:3306` or `*:3306`

### 5.3 Verify database creation and grant assignment

**Command:** `mariadb -e "SHOW DATABASES; SELECT User, Host FROM mysql.user WHERE User='app_user';"`

**What it checks and variables to look for:**

- **Database**: Output list must include `vpn_db`
- **User**: Must display `app_user`
- **Host**: Must display `10.10.20.%`

### 5.4 Verify user database access and password authentication

**Command:** `mariadb -u app_user -h 127.0.0.1 -p'SecureAppPassword123!' -D vpn_db -e "SELECT DATABASE();"`

**What it checks and variables to look for:**

- **DATABASE()**: Must return `vpn_db`

<!-- Created by: Gergő Téringer, 2026 -->