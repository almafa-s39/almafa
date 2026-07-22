<!-- 
---
title: "Squid Transparent Proxy"
author: "Gergő Téringer"
---
 -->
# Squid Transparent Proxy

This document provides administrative procedures for configuring the Squid caching and forwarding web proxy on Debian 13 (Trixie). It covers transparent and non-transparent operational modes, TLS encrypted traffic inspection (SSL Bumping), custom access control lists (ACLs), and firewall redirection.

> [!NOTE]
> To inspect TLS-encrypted traffic, the proxy must dynamically generate certificates on-the-fly using a local CA. This requires the SSL-enabled Squid package rather than the default distribution package.

## 1. Package Installation

Ensure any existing standard Squid packages are removed before installing the OpenSSL-compiled version required for deep packet inspection.

```Bash
# Remove the standard squid package if it is installed
apt remove squid

# Install the OpenSSL-enabled squid package
apt install squid-openssl
```

**Command Breakdown & Explanation:**

- `apt remove squid`: Prevents package conflicts with the standard non-SSL version.
- `apt install squid-openssl`: Installs the specialized binary compiled with `--with-openssl` and `--enable-ssl-crtd` options.

## 2. Certificate and SSL Database Initialization

To inspect TLS traffic, the proxy must act as a Man-in-the-Middle (MITM). It reads the destination's certificate and generates a forged certificate on-the-fly signed by your local Proxy CA. You must create this CA, generate Diffie-Hellman parameters, and initialize the SSL database.

> [!WARNING]
> On Debian systems, the Squid daemon runs under the `proxy` user and group. All certificate files and databases must be owned by this user to function correctly.

```Bash
# Create the directory to store the Proxy CA certificates
mkdir -p /etc/squid/cert
cd /etc/squid/cert

# Generate the self-signed Proxy CA certificate and private key
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
  -keyout proxy.key -out proxy.crt -subj "/C=HU/O=Kontozo/CN=Squid Proxy CA"

# Generate Diffie-Hellman parameters
openssl dhparam -out dhparam.pem 2048

# Secure the certificate directory permissions
chown -R proxy:proxy /etc/squid/cert
chmod -R 400 /etc/squid/cert

# Initialize the dynamic SSL certificate database and set ownership
/usr/lib/squid/security_file_certgen -c -s /var/lib/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/ssl_db
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `security_file_certgen`: The helper utility that creates and manages the internal database where Squid temporarily stores the on-the-fly generated host certificates.

### 3. External ACL and Error Page Setup

Squid can read Access Control Lists (ACLs) from external text files, making large lists easier to manage. You can also map custom error pages to specific ACL rejections.

```Bash
# Create an external ACL file containing streaming video MIME types
cat << 'EOF' > /etc/squid/video-mime.txt
^video/x-ms-asf$
^application/vnd.ms.wms-hdr.asfv1$
^application/x-mms-framed$
^application/vnd.yt-ump$
^application/vnd.apple.mpegurl$
EOF

# Create a custom error page for blocked domains
echo "ERR_ACCESS_DENIED: Domain blocked by corporate policy." > /usr/share/squid/errors/en/CUSTOM_ERROR
```

## 4. Main Squid Configuration

Backup the default configuration file and create a new `/etc/squid/squid.conf`. This unified configuration file defines transparent intercept ports, non-transparent ports, TLS inspection (SSL Bump) directives, and the ACL filtering logic.

> [!TIP]
> To disable decryption and only filter based on TCP/TLS metadata (like SNI), change `ssl_bump bump all` to `ssl_bump splice all`.

```Bash
# Backup the original configuration
mv /etc/squid/squid.conf /etc/squid/squid.conf.backup

# Write the new unified configuration file
cat << 'EOF' > /etc/squid/squid.conf
# -------------------------------------------------------------
# 1. Listening Ports
# -------------------------------------------------------------
# Regular non-transparent HTTP proxy port
http_port 3128

# Transparent HTTP intercept port
http_port 3129 intercept

# Transparent HTTPS intercept port with SSL Bumping
https_port 3130 intercept ssl-bump \
  generate-host-certificates=on \
  dynamic_cert_mem_cache_size=4MB \
  tls-cert=/etc/squid/cert/proxy.crt \
  tls-key=/etc/squid/cert/proxy.key \
  tls-dh=/etc/squid/cert/dhparam.pem \
  tls-default-ca=on \
  options=NO_SSLv3,NO_TLSv1,NO_TLSv1_1

# Non-transparent HTTPS port for explicit client configuration
http_port 3131 ssl-bump \
  generate-host-certificates=on \
  dynamic_cert_mem_cache_size=4MB \
  tls-cert=/etc/squid/cert/proxy.crt \
  tls-key=/etc/squid/cert/proxy.key \
  tls-dh=/etc/squid/cert/dhparam.pem \
  tls-default-ca=on \
  options=NO_SSLv3,NO_TLSv1,NO_TLSv1_1

# -------------------------------------------------------------
# 2. SSL Bump and Certificate Generation
# -------------------------------------------------------------
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/ssl_db -M 4MB
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# -------------------------------------------------------------
# 3. Access Control Lists (ACLs)
# -------------------------------------------------------------
acl CONNECT method CONNECT
acl blockdomains dstdomain .index.hu .fidesz.hu
acl streamreq req_mime_type -i "/etc/squid/video-mime.txt"
acl streamrep rep_mime_type -i "/etc/squid/video-mime.txt"

# -------------------------------------------------------------
# 4. Access Policies and Error Pages
# -------------------------------------------------------------
# Block explicit domains and assign the custom error page
http_access deny blockdomains CONNECT
http_reply_access deny blockdomains
deny_info CUSTOM_ERROR blockdomains

# Block video streaming in both directions
http_access deny streamreq
http_reply_access deny streamrep

# Allow all other traffic
http_access allow all

# -------------------------------------------------------------
# 5. Miscellaneous Configurations
# -------------------------------------------------------------
reply_header_add x-secured-by "clearsky-proxy"
shutdown_lifetime 5 seconds
EOF

# Restart the Squid service to apply the configuration
systemctl restart squid
```

**Command Breakdown & Explanation:**

Before lists place a blank line!

- `intercept`: Informs Squid that traffic arriving on this port was NAT-redirected by the firewall, requiring special transparent handling.
- `ssl-bump`: Activates the TLS inspection engine.
- `tls-default-ca=on`: Forces Squid to trust the system CA certificates (e.g., from `/usr/local/share/ca-certificates/`). Without this, locally trusted sites will cause certificate errors in the client browser.
- `shutdown_lifetime 5 seconds`: Reduces the default 30-second graceful shutdown delay when restarting the daemon.

## 5. Transparent Proxy Firewall Redirection

For transparent proxying, the client devices do not know the proxy exists. You must utilize `nftables` on your router/firewall to silently capture web traffic and forcefully redirect it to the Squid interception ports.

> [!WARNING]
> Ensure you redirect packets to the `intercept` listeners defined in your config (ports 3129 and 3130). Do not redirect traffic to the standard 3128 port.

```Bash
# Example snippet for /etc/nftables.conf on the proxy host
cat << 'EOF' >> /etc/nftables.conf

table inet filter {
  chain portfw {
    type nat hook prerouting priority dstnat;
    
    # Redirect HTTP traffic to the HTTP intercept port
    ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 80 redirect to 3129;
    
    # Redirect HTTPS traffic to the HTTPS intercept port
    ip saddr { 10.1.10.0/24, 10.1.30.0/24 } tcp dport 443 redirect to 3130;
  }
}
EOF

# Reload the nftables ruleset
nft -f /etc/nftables.conf
```

## 6. Non-Transparent Client Configurations

If you are utilizing the explicit, non-transparent port (3131), clients must be manually configured to route traffic to the proxy server IP (`10.1.30.1` in this example).

### 6.1 APT Package Manager Proxy

```Bash
# Enforce proxy usage for APT operations globally
cat << 'EOF' > /etc/apt/apt.conf.d/80proxy
Acquire::http::proxy "http://10.1.30.1:3131";
Acquire::https::proxy "http://10.1.30.1:3131";
EOF
```

### 6.2 Wget Proxy Configuration

```Bash
# Enforce proxy usage for wget globally
cat << 'EOF' >> /etc/wgetrc
http_proxy=10.1.30.1:3131
https_proxy=10.1.30.1:3131
EOF
```

## 7. Verification and Troubleshooting

> [!NOTE]
> Validate the configuration syntax, service state, and listening ports on Debian 13 using standard administrative tools. Client browsers must also be configured to explicitly trust your `proxy.crt` file.

### 7.1 Verify Squid configuration syntax

**Command:** `squid -k parse`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Output**: Must parse the configuration without throwing `FATAL` or `ERROR` messages regarding missing certificates, undefined ACLs, or syntax typos.

### 7.2 Verify Squid service status

**Command:** `systemctl status squid`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/squid.service; enabled)`

### 7.3 Verify network listening state on configured ports

**Command:** `ss -tulnp | grep squid`

**What it checks and variables to look for:**

Before lists place a blank line!

- **Ports**: Must display `LISTEN` on ports `3128`, `3129`, `3130`, and `3131` according to your `squid.conf` file.

<!-- Created by: Gergő Téringer, 2026 -->