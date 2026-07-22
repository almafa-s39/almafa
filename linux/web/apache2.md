<!-- 
---
title: "Apache2 Web Server"
author: "Gergő Téringer"
---
 -->
# Apache2 Web Server

This document provides administrative procedures for configuring the Apache2 web server on Debian 13 (Trixie). It covers enforcing HTTP-to-HTTPS redirection, enabling Mutual TLS (mTLS) for client certificate authentication, and integrating LDAP for centralized directory authentication.

> [!NOTE]
> Apache2 requires specific dynamic modules to handle SSL/TLS, URL rewriting, and LDAP authentication. These modules must be explicitly enabled using the `a2enmod` utility before the configuration will parse correctly.

## 1. Package Installation and Module Enablement

Install the Apache2 web server and enable the required underlying modules for SSL, rewriting, and LDAP authentication.

```Bash
# Install the core Apache2 server
apt install apache2

# Enable the required Apache2 modules
a2enmod ssl rewrite ldap authnz_ldap

# Restart the service to load the newly enabled modules
systemctl restart apache2
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `apt install apache2`: Installs the core web server daemon and creates the standard `/etc/apache2/` directory structure.
- `a2enmod ssl`: Enables the `mod_ssl` engine required for HTTPS and mTLS.
- `a2enmod rewrite`: Enables `mod_rewrite`, allowing dynamic HTTP-to-HTTPS URL redirection.
- `a2enmod ldap authnz_ldap`: Enables the modules required to connect to an LDAP directory and process Basic Authentication requests against it.

## 2. Virtual Host Configuration

Apache2 utilizes Virtual Host files to define sites. We will create a unified configuration file that handles the port 80 redirection and the secure port 443 site containing the mTLS and LDAP directives.

> [!IMPORTANT]
> The client certificate verification (mTLS) happens at the TLS handshake layer, while the LDAP authentication happens at the HTTP layer. Both must succeed for the user to access the `<Location />` block.

```Bash
# Create the secure virtual host configuration file
cat << 'EOF' > /etc/apache2/sites-available/secure-site.conf
# HTTP Server Block: Redirect all traffic to HTTPS
<VirtualHost *:80>
    ServerName secure.company.com
    ServerAlias www.secure.company.com

    # Enable the rewrite engine and forcefully redirect to HTTPS
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
    
    ErrorLog ${APACHE_LOG_DIR}/redirect_error.log
    CustomLog ${APACHE_LOG_DIR}/redirect_access.log combined
</VirtualHost>

# HTTPS Server Block: SSL, Client Cert Auth, and LDAP Auth
<VirtualHost *:443>
    ServerName secure.company.com
    DocumentRoot /var/www/html

    # 1. Server SSL/TLS Certificate Configuration
    SSLEngine on
    SSLCertificateFile /ca/server.crt
    SSLCertificateKeyFile /ca/server.key

    # 2. Client Certificate Authentication (mTLS)
    # The CA used to verify the client's submitted certificate
    SSLCACertificateFile /ca/ca.crt
    # Enforce that the client MUST present a valid certificate
    SSLVerifyClient require
    SSLVerifyDepth 2

    # 3. Directory and LDAP Authentication Configuration
    <Location />
        # Require Basic Auth utilizing the LDAP provider
        AuthType Basic
        AuthName "Restricted Access - Enter LDAP Credentials"
        AuthBasicProvider ldap

        # Define the LDAP connection string and search filters
        AuthLDAPURL "ldap://10.1.10.10:389/dc=company,dc=com?uid?sub?(objectClass=person)"
        
        # Define the service account used to bind and query the LDAP directory
        AuthLDAPBindDN "cn=admin,dc=company,dc=com"
        AuthLDAPBindPassword "SecretPassword"

        # Enforce that the matched LDAP user must successfully authenticate
        Require valid-user
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/secure_error.log
    CustomLog ${APACHE_LOG_DIR}/secure_access.log combined
</VirtualHost>
EOF
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]`: Captures the exact URL string the user typed in HTTP and bounces their browser to the exact same path using a 301 Permanent Redirect to HTTPS.
- `SSLVerifyClient require`: Drops the connection immediately during the TLS handshake if the client's browser does not provide a valid certificate signed by the CA specified in `SSLCACertificateFile`.
- `SSLVerifyDepth 2`: Allows intermediate Subordinate CAs to exist in the trust chain (up to a depth of 2) when validating the client's certificate.
- `AuthLDAPURL`: Defines the LDAP endpoint protocol, IP/Port, Base DN (`dc=company,dc=com`), login attribute (`uid`), search scope (`sub`), and an optional object filter (`objectClass=person`).

## 3. Site Activation and Reload

After writing the configuration file, you must disable the default unencrypted site, enable your new secure site, and reload the Apache2 daemon.

```Bash
# Disable the default HTTP-only site provided by Debian
a2dissite 000-default.conf

# Enable the newly created secure site
a2ensite secure-site.conf

# Test the configuration syntax for errors
apache2ctl configtest

# Reload the Apache2 daemon to apply changes seamlessly
systemctl reload apache2
```

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the Apache2 configuration syntax, service status, and active listening ports on Debian 13 using standard administrative tools.

### 4.1 Verify Apache2 configuration syntax

**Command:** `apache2ctl configtest`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Syntax output**: Must explicitly return `Syntax OK`. If there are missing modules, malformed tags, or unreadable certificate paths, the engine will output the exact line number causing the fatal error.

### 4.2 Verify Apache2 service status

**Command:** `systemctl status apache2`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/apache2.service; enabled)`

### 4.3 Verify network listening state on configured ports

**Command:** `ss -tulnp | grep apache2`

**What it checks and variables to look for:**

Before lists place a blank line!

- **HTTP/HTTPS**: Must display `LISTEN` on `*:80` and `*:443` (or `0.0.0.0:80` / `0.0.0.0:443`).

<!-- Created by: Gergő Téringer, 2026 -->