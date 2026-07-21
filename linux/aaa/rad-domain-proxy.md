<!-- 
---
title: "FreeRADIUS LDAP Domain Proxy"
author: "Gergő Téringer"
---
 -->
# FreeRADIUS LDAP Domain Proxy

This document provides administrative procedures for deploying FreeRADIUS on Debian 13 (Trixie) configured as both a local LDAP-authenticating server and a multi-domain RADIUS proxy. Requests targeting the local domain (`example.net`) or containing no realm (`NULL`) are authenticated against OpenLDAP, while requests targeting `site.example.net` are proxied to an upstream RADIUS server.

> [!WARNING]
> If you plan to allocate dynamic IP pools using the `Rad-Pool` attribute with OpenVPN, note that the standard `openvpn-auth-radius` plugin does not process `Rad-Pool` response attributes out-of-the-box. A custom management script or wrapper is required to parse and apply pool assignments.

## 1. LDAP Directory Structure and Schema Requirements

Before integrating FreeRADIUS with LDAP, the OpenLDAP directory must contain valid user entries (`posixAccount` / `inetOrgPerson`) and group records (`posixGroup`). User identity mapping in this architecture uses the `mail` attribute (e.g., `username@example.net`) as the primary RADIUS login identifier.

```ldif
# User record structure in OpenLDAP
dn: uid=username,ou=ou,dc=example,dc=net
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
uid: username
givenName: username
sn: username
cn: username
uidNumber: 11001
gidNumber: 11001
mail: username@example.net
homeDirectory: /home/username
loginShell: /bin/bash
userPassword: {SSHA}YourHashedPasswordString

# Group record structure in OpenLDAP
dn: cn=group,ou=ou,dc=example,dc=net
objectClass: posixGroup
objectClass: shadowGroup
cn: group
gidNumber: 11001
memberUid: username
```

**Command Breakdown & Explanation:**

- `mail: username@example.net`: Used as the lookup key during RADIUS authentication requests.
- `userPassword`: Stores the user credential (commonly SSHA-hashed), requiring FreeRADIUS to handle Password Authentication Protocol (PAP) processing.
- `objectClass: posixGroup`: Specifies the group schema type.
- `memberUid: username`: Attributes mapping group membership by linking to the user's `uid` field.

## 2. FreeRADIUS Package Installation and Client Authorization

FreeRADIUS module binaries and administrative tools are distributed across separate Debian packages. Initial configuration requires installing these packages and authorizing Network Access Servers (NAS clients) such as VPN gateways.

> [!NOTE]
> FreeRADIUS configuration files on Debian 13 reside in `/etc/freeradius/3.0/`.

```bash
# Update repositories and install FreeRADIUS with LDAP support tools
apt install freeradius freeradius-ldap freeradius-utils

# Remove the default inner-tunnel configuration to ensure a clean virtual server setup
rm -f /etc/freeradius/3.0/sites-enabled/inner-tunnel

# Add NAS Client definition for the OpenVPN Server
tee -a /etc/freeradius/3.0/clients.conf > /dev/null << 'EOF'

client vpn.example.net {
    ipaddr = 10.0.0.1
    secret = YourPassword
}
EOF
```

**Command Breakdown & Explanation:**

- `apt install freeradius freeradius-ldap freeradius-utils`: Installs the core RADIUS daemon, the LDAP module extension (`rlm_ldap`), and diagnostic testing utilities (`radtest`).
- `rm -f .../sites-enabled/inner-tunnel`: Removes default EAP inner-tunnel bindings that are unneeded for direct PAP/LDAP proxy deployments.
- `client vpn.example.net`: Defines the OpenVPN gateway (`10.0.0.1`) as a trusted client authorized to send authentication requests using a shared secret key.

## 3. Realm Proxy Configuration

The `proxy.conf` file manages request routing based on the domain suffix contained in the `User-Name` attribute (e.g., `user@site.example.net`).

> [!IMPORTANT]
> The `nostrip` option prevents FreeRADIUS from stripping the `@site.example.net` realm suffix from the packet payload prior to forwarding it to the upstream server.

```bash
# Append home server definitions and realm routing rules to proxy.conf
tee -a /etc/freeradius/3.0/proxy.conf > /dev/null << 'EOF'

home_server radius.site.example.net {
    type     = auth
    ipaddr   = 172.16.0.1
    port     = 1812
    secret   = YourPassword
}

home_server_pool pool_site {
    type        = load-balance
    home_server = radius.site.example.net
}

realm site.example.net {
    auth_pool = pool_site
    nostrip
}

realm example.net {
}

realm NULL {
}
EOF
```

**Command Breakdown & Explanation:**

- `home_server radius.site.example.net`: Defines the remote upstream RADIUS server IP (`172.16.0.1`) and port (`1812`).
- `home_server_pool pool_site`: Groups upstream targets into a load-balancing pool.
- `realm site.example.net`: Directs any request ending in `@site.example.net` to `pool_site`.
- `realm example.net` / `realm NULL`: Directs requests matching the local domain or containing no domain realm to local processing pipelines.

## 4. Custom Virtual Server Configuration

To control request evaluation cleanly, remove default site bindings and provision a dedicated virtual server that evaluates domain realms, queries LDAP, handles PAP authentication, and checks LDAP group membership during the `post-auth` phase.

```bash
# Remove all default sites-enabled symlinks
rm -f /etc/freeradius/3.0/sites-enabled/*

# Create a new custom default virtual server configuration
tee /etc/freeradius/3.0/sites-available/custom-default > /dev/null << 'EOF'
server default {
    listen {
        type   = auth
        port   = 1812
        ipaddr = *
    }

    listen {
        type   = acct
        port   = 1813
        ipaddr = *
    }

    authorize {
        suffix

        if (Realm == "example.net" || Realm == "NULL") {
            ldap
        }
        pap
    }

    authenticate {
        Auth-Type PAP {
            pap
        }
        Auth-Type LDAP {
            ldap
        }
    }

    post-auth {
        if (LDAP-Group == "cn=group,ou=ou,dc=example,dc=net") {
            update-reply {
                Reply-Message := "Access Granted: Group Verified"
            }
        }
    }
}
EOF

# Enable the custom virtual server
ln -sf /etc/freeradius/3.0/sites-available/custom-default /etc/freeradius/3.0/sites-enabled/default
```

**Command Breakdown & Explanation:**

- `suffix`: Parses the realm from the incoming `User-Name` attribute based on rules in `proxy.conf`.
- `if (Realm == "example.net" || Realm == "NULL")`: Ensures LDAP database queries occur only for local or realm-less users, allowing proxied realms (`site.example.net`) to pass directly to the proxy module.
- `pap`: Computes hashes required to validate passwords against stored LDAP passwords.
- `post-auth`: Executes after successful authentication. The `LDAP-Group` check queries the directory to verify whether the user belongs to the specified group DN before returning response attributes.

## 5. LDAP Module Activation and Configuration

To connect FreeRADIUS to OpenLDAP, enable the LDAP module symlink and adjust the connection parameters, user search filters, and group membership attributes.

```bash
# Enable the LDAP module in mods-enabled
ln -sf /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/ldap

# Configure module parameters in /etc/freeradius/3.0/mods-enabled/ldap
tee /etc/freeradius/3.0/mods-enabled/ldap > /dev/null << 'EOF'
ldap {
    server = 'ldap://ldap.example.net'
    port = 389
    identity = 'cn=admin,dc=example,dc=net'
    password = YourPassword
    base_dn = 'dc=example,dc=net'

    user {
        base_dn = 'dc=example,dc=net'
        filter = "(mail=%{User-Name})"
        scope = 'sub'
    }

    group {
        base_dn = 'dc=example,dc=net'
        scope = 'sub'
        membership_attribute = 'memberUid'
    }

    # Optional TLS settings if using Secure LDAP (ldaps / StartTLS)
    # tls {
    #     ca_file = /etc/ssl/certs/ca-certificates.crt
    #     require_cert = 'demand'
    # }
}
EOF
```

**Command Breakdown & Explanation:**

- `server = 'ldap://ldap.example.net'`: Destination hostname or IP address of the OpenLDAP server.
- `identity` / `password`: Bind credentials used by FreeRADIUS to search the directory structure.
- `filter = "(mail=%{User-Name})"`: Maps incoming RADIUS identity strings against the user's LDAP `mail` attribute.
- `membership_attribute = 'memberUid'`: Configures group lookups to check the `memberUid` attribute in `posixGroup` records.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate local authentication, realm proxying, and LDAP connectivity on Debian 13 using standard diagnostic tools.

### 6.1 Verify FreeRADIUS service status and port bindings

**Command:** `systemctl status freeradius`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/freeradius.service; enabled)`

### 6.2 Verify local realm LDAP authentication and group response

**Command:** `radtest username@example.net "YourPassword" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must be `Access-Accept`
- **Reply-Message**: Must contain `Access Granted: Group Verified`

### 6.3 Verify proxy realm forwarding to upstream home server

**Command:** `radtest username@site.example.net "YourPassword" 127.0.0.1 0 testing123`

**What it checks and variables to look for:**

- **Received response**: Must reflect the response returned by server `172.16.0.1` (e.g., `Access-Accept` or `Access-Reject`)

### 6.4 Verify direct OpenLDAP service binding and user search

**Command:** `ldapsearch -x -H ldap://ldap.example.net -D "cn=admin,dc=example,dc=net" -w "YourPassword" -b "dc=example,dc=net" "(mail=username@example.net)"`

**What it checks and variables to look for:**

- **result**: Must show `0 Success`
- **numEntries**: Must return `1` matching record containing valid `uid` and `mail` attributes

<!-- Created by: Gergő Téringer, 2026 -->