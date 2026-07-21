<!-- 
---
title: "RADIUS SQL Settings (LowBudget ISE)"
author: "Gergő Téringer"
---
 -->
# RADIUS SQL Settings (LowBudget ISE)

This document details administrative procedures for configuring FreeRADIUS on Debian 13 (Trixie) to perform dynamic IP allocation and client certificate authorization via a MySQL database. In this architecture, FreeRADIUS queries a MySQL database (`vpn.users`) using the client's X.509 certificate Common Name (CN) passed in the `User-Name` attribute, retrieves the assigned static IP address, injects it into the `Framed-IP-Address` reply attribute, and authorizes the session.

> [!NOTE]
> FreeRADIUS configuration files reside in `/etc/freeradius/3.0/`. On Debian 13, database connectivity requires the `freeradius-mysql` package and activation of the `rlm_sql` module.

## 1. Package Installation and Base Environment Setup

> [!IMPORTANT]
> Disabling default EAP inner-tunnel configurations prevents startup port conflicts and simplifies packet processing when using direct SQL-driven authorization.

```bash
# Install FreeRADIUS core binaries, MySQL module driver, and diagnostic tools
apt install freeradius freeradius-mysql freeradius-utils

# Remove default inner-tunnel site symlink
rm -f /etc/freeradius/3.0/sites-enabled/inner-tunnel
```

**Command Breakdown & Explanation:**

- `apt install freeradius freeradius-mysql freeradius-utils`: Installs the core daemon, the `rlm_sql_mysql` driver package, and testing tools (`radtest`).
- `rm -f /etc/freeradius/3.0/sites-enabled/inner-tunnel`: Removes unused default EAP inner-tunnel virtual server configurations.

## 2. Virtual Server SQL Authorization Pipeline Configuration

To enable dynamic IP lookups, update the `default` virtual server policy in `/etc/freeradius/3.0/sites-available/default`. The `authorize` section queries the MySQL database using xlat expansion (`%{sql:...}`), populates control variable `&Tmp-String-0` with the returned IP address string, and assigns `&Framed-IP-Address` if a record exists.

> [!TIP]
> If the SQL lookup returns an empty string or fails, the user request is immediately rejected.

```bash
# Provision the default virtual server with embedded SQL query authorization
tee /etc/freeradius/3.0/sites-available/default > /dev/null << 'EOF'
server default {
    listen {
        type   = auth
        ipaddr = *
        port   = 1812
    }

    listen {
        type   = acct
        ipaddr = *
        port   = 1813
    }

    authorize {
        preprocess

        # Look up the certificate CN (sent as User-Name) in the vpn.users table
        update control {
            &Tmp-String-0 := "%{sql:SELECT ip FROM users WHERE username = '%{User-Name}'}"
        }

        if (&control:Tmp-String-0 && &control:Tmp-String-0 != "") {
            update reply {
                &Framed-IP-Address := "%{control:Tmp-String-0}"
            }
            update control {
                &Auth-Type := Accept
            }
        }
        else {
            reject
        }
    }

    accounting {
        ok
    }
}
EOF

# Ensure the default virtual server is symlinked in sites-enabled
ln -sf /etc/freeradius/3.0/sites-available/default /etc/freeradius/3.0/sites-enabled/default
```

**Command Breakdown & Explanation:**

- `preprocess`: Formats incoming attributes prior to database evaluation.
- `update control { &Tmp-String-0 := "%{sql:SELECT ip FROM users WHERE username = '%{User-Name}'}" }`: Executes an inline SQL query matching the incoming `User-Name` against the `username` column, storing the result in temporary buffer `&Tmp-String-0`.
- `if (&control:Tmp-String-0 && &control:Tmp-String-0 != "")`: Validates that a non-empty IP address record was returned from the database.
- `update reply { &Framed-IP-Address := "%{control:Tmp-String-0}" }`: Injects the IP address string into the RADIUS `Access-Accept` response payload for network client assignment.
- `&Auth-Type := Accept`: Directly approves authentication without requiring password verification, trusting the pre-validated X.509 client certificate identity.
- `reject`: Rejects client access requests if no matching user record exists in the database.

## 3. Network Access Server Client Authorization

Authorized network access devices (such as OpenVPN endpoints) must be explicitly registered in `/etc/freeradius/3.0/clients.conf` with a shared secret key.

```bash
# Add OpenVPN gateway client entry to clients.conf
tee -a /etc/freeradius/3.0/clients.conf > /dev/null << 'EOF'

client openvpn_server {
    ipaddr    = 10.10.10.254
    secret    = Passw0rd!
    shortname = ovpn
}
EOF
```

**Command Breakdown & Explanation:**

- `client openvpn_server`: Creates a distinct client configuration block for the VPN gateway.
- `ipaddr = 10.10.10.254`: Identifies the source IP address sending RADIUS access requests.
- `secret = Passw0rd!`: Defines the shared secret string used to encrypt payload data between OpenVPN and FreeRADIUS.

## 4. MySQL Module Configuration and Service Activation

Configure the `rlm_sql` module parameters in `/etc/freeradius/3.0/mods-available/sql` to specify database host location, database name (`vpn`), and database authentication credentials.

> [!CAUTION]
> Ensure the MySQL user (`radius`) has `SELECT` permissions on the `vpn.users` table and that MySQL firewall rules permit TCP port 3389/3306 communication from the FreeRADIUS server IP.

```bash
# Configure the MySQL module settings
tee /etc/freeradius/3.0/mods-available/sql > /dev/null << 'EOF'
sql {
    dialect = "mysql"
    driver = "rlm_sql_${dialect}"

    server = "10.10.20.10"
    port = 3306
    login = "radius"
    password = "Passw0rd!"
    radius_db = "vpn"
    read_client = no

    mysql {
        # Optional TLS parameters for encrypted database connections
        # tls {
        # }
    }
}
EOF

# Enable the SQL module symlink in mods-enabled
ln -sf /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/sql

# Restart FreeRADIUS to apply pipeline and module configurations
systemctl restart freeradius
```

**Command Breakdown & Explanation:**

- `dialect = "mysql"` / `driver = "rlm_sql_mysql"`: Instructs FreeRADIUS to utilize the MySQL database driver.
- `server = "10.10.20.10"`: IP address of the target MySQL database host.
- `login = "radius"` / `password = "Passw0rd!"`: Credentials used by FreeRADIUS to bind to the MySQL database engine.
- `radius_db = "vpn"`: Database catalog storing the `users` table.
- `read_client = no`: Disables fetching client definitions dynamically from the database to improve performance.
- `ln -sf .../mods-available/sql .../mods-enabled/sql`: Enables the SQL module in active runtime memory.
- `systemctl restart freeradius`: Reloads all virtual server, client, and module definitions.

## 5. Latest step: [OpenVPN setup](/linux/vpn/ovpn-ise.md)

## 6. Next step: [Database setup](/linux/db/ise.md)

## 7. Verification and Troubleshooting

> [!NOTE]
> Validate FreeRADIUS daemon state, database query evaluation, and response attributes on Debian 13 using standard diagnostic commands.

### 7.1 Verify FreeRADIUS service running status

**Command:** `systemctl status freeradius`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/freeradius.service; enabled)`

### 7.2 Verify MySQL database connection and network reachability

**Command:** `mariadb -h 10.10.20.10 -u radius -p'Passw0rd!' -D vpn -e "SELECT ip FROM users LIMIT 1;"`

**What it checks and variables to look for:**

- **ERROR**: Must not return connection errors or access denied messages
- **ip**: Must execute successfully and display matching table output

### 7.3 Verify SQL user lookup and Framed-IP response using radtest

**Command:** `radtest cert_user_01 "" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Accept`
- **Framed-IP-Address**: Must match the IP assigned in the `vpn.users` table (e.g., `10.8.0.50`)
- **Code**: Must be `2`

### 7.4 Verify access rejection for non-existent certificate identities

**Command:** `radtest non_existent_user "" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Reject`
- **Code**: Must be `3`

<!-- Created by: Gergő Téringer, 2026 -->