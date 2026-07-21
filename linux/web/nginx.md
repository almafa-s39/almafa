<!-- 
---
title: "Nginx"
author: "Gergő Téringer"
---
 -->
# Nginx Web Server

This document provides administrative procedures for configuring the NGINX web server on Debian 13 (Trixie). It covers enforcing HTTP to HTTPS redirection, enabling Mutual TLS (mTLS) for client certificate authentication, and integrating LDAP for centralized user authentication.

> [!NOTE]
> Native open-source NGINX requires an additional dynamic module to support LDAP authentication directly. We will install the Debian-provided LDAP authentication module alongside the core web server.

## 1. Package Installation

Install the NGINX web server and the specific HTTP LDAP authentication module.

```Bash
# Install the core NGINX server and the LDAP authentication module
apt install nginx libnginx-mod-http-auth-ldap

# Enable the service to start at boot and launch it immediately
systemctl enable nginx --now
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `apt install nginx`: Installs the core web server daemon.
- `libnginx-mod-http-auth-ldap`: Installs the dynamic module required to use the `ldap_server` and `auth_ldap` configuration directives directly within NGINX.

## 2. Global Configuration and Virtual Hosts

NGINX uses server blocks (virtual hosts) to manage listening ports and routing. The following configuration overwrites the default site to enforce HTTP-to-HTTPS redirection, require a trusted client certificate (mTLS), and prompt for LDAP credentials.

> [!WARNING]
> The `ldap_server` block must be defined within the `http` context. Since Debian includes files from `sites-enabled/` directly into the `http` block of `nginx.conf`, we can place it at the top of our site configuration file.

```Bash
# Overwrite the default NGINX site configuration
cat << 'EOF' > /etc/nginx/sites-available/default
# Define the LDAP server connection parameters
ldap_server my_ldap_server {
    url ldap://10.1.10.10:389/dc=company,dc=com?uid?sub?(objectClass=person);
    binddn "cn=admin,dc=company,dc=com";
    binddn_passwd "SecretPassword";
    group_attribute memberUid;
    group_attribute_is_dn off;
    require valid_user;
}

# HTTP Server Block: Redirect all traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    
    # Matches any requested hostname. Change this to the specific FQDN in production.
    server_name _; 

    # Issue a 301 Permanent Redirect to the HTTPS equivalent of the requested URL
    return 301 https://$host$request_uri;
}

# HTTPS Server Block: SSL, Client Cert Auth, and LDAP Auth
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    
    server_name _; 
    root /var/www/html;
    index index.html;

    # 1. Server SSL/TLS Certificate Configuration
    ssl_certificate /ca/server.crt;
    ssl_certificate_key /ca/server.key;

    # 2. Client Certificate Authentication (mTLS)
    # The CA used to verify the client's submitted certificate
    ssl_client_certificate /ca/ca.crt;
    # Enforce that the client MUST present a valid certificate
    ssl_verify_client on; 

    location / {
        # 3. LDAP Authentication
        # The prompt displayed to the user in their browser
        auth_ldap "Restricted Access - Enter LDAP Credentials";
        # Points to the ldap_server block defined at the top of the file
        auth_ldap_servers my_ldap_server;

        try_files $uri $uri/ =404;
    }
}
EOF

# Test the configuration syntax and reload the NGINX daemon
nginx -t
systemctl reload nginx
```

**Command Breakdown & Explanation:**

- `url ldap://...`: Defines the LDAP endpoint, the base Search DN (`dc=company,dc=com`), the attribute used for usernames (`uid`), the search scope (`sub`), and the filter (`objectClass=person`).
- `return 301 https://$host$request_uri`: Captures the exact URL string the user typed in HTTP and bounces their browser to the exact same path using HTTPS.
- `ssl_client_certificate /ca/ca.crt`: Informs NGINX which Root/Subordinate CA is trusted to validate the certificates presented by incoming clients.
- `ssl_verify_client on`: Drops the connection immediately during the TLS handshake if the client's browser does not provide a valid certificate signed by the designated CA.
- `nginx -t`: Parses the configuration files to check for missing semicolons, invalid directives, or certificate path errors before applying changes.

## 3. Verification and Troubleshooting

> [!NOTE]
> Validate the NGINX configuration syntax, service status, and active listening ports on Debian 13 using standard administrative tools.

### 3.1 Verify NGINX configuration syntax

**Command:** `nginx -t`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Syntax output**: Must explicitly return `nginx: configuration file /etc/nginx/nginx.conf syntax is ok` and `nginx: configuration file /etc/nginx/nginx.conf test is successful`.

### 3.2 Verify NGINX service status

**Command:** `systemctl status nginx`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/nginx.service; enabled)`

### 3.3 Verify network listening state on configured ports

**Command:** `ss -tulnp | grep nginx`

**What it checks and variables to look for:**

Before lists place a blank line!

- **HTTP/HTTPS**: Must display `LISTEN` on `*:80` and `*:443` (or `0.0.0.0:80` / `0.0.0.0:443`).

<!-- Created by: Gergő Téringer, 2026 -->