<!-- 
---
title: "HaProxy"
author: "Gergő Téringer"
---
 -->
# HaProxy

HAProxy Load Balancer and Reverse Proxy Guide
Markdown

This document provides administrative procedures for configuring HAProxy as a reverse proxy and load balancer on Debian 13 (Trixie). It covers SSL/TLS termination, HTTP-to-HTTPS redirection, Access Control List (ACL) routing based on source IPs and domain Host headers, and backend health monitoring.

> [!NOTE]
> HAProxy processes traffic at Layer 4 (TCP) and Layer 7 (HTTP). To inspect and route based on HTTP headers (like domains), the traffic must be decrypted at the frontend using SSL termination, or passed through via SNI at Layer 4.

## 1. Package Installation and Certificate Preparation

HAProxy requires a combined PEM file for SSL termination that includes both the server certificate and its unencrypted private key, along with the CA chain if applicable.

```Bash
# Install the HAProxy package
apt install haproxy

# Combine the server certificate and private key into a single PEM file for HAProxy
cat /ca/server.crt /ca/server.key > /ca/server.pem

# Restrict permissions on the combined file since it contains the private key
chmod 600 /ca/server.pem
```

**Command Breakdown & Explanation:**

- `apt install haproxy`: Installs the core daemon. The main configuration file is located at `/etc/haproxy/haproxy.cfg`.
- `cat ... > /ca/server.pem`: Concatenates the public certificate and private key. HAProxy expects the `crt` parameter to point to a file containing both.

## 2. Frontend Configuration (Routing and SSL)

The `frontend` block defines how HAProxy listens for incoming connections. This configuration combines port 80 and 443 listeners, enforces HTTPS redirection, and utilizes ACLs to route traffic based on the requester's IP subnet and the requested domain name.

> [!TIP]
> The `option forwardfor` directive is critical when HAProxy is acting as a reverse proxy. It injects the `X-Forwarded-For` HTTP header, ensuring backend web servers log the client's actual IP address rather than the HAProxy server's IP.

```Bash
# Append the frontend configuration to /etc/haproxy/haproxy.cfg
cat << 'EOF' >> /etc/haproxy/haproxy.cfg

frontend web-in
    # Listen on HTTP and HTTPS
    bind *:80
    bind *:443 ssl crt /ca/server.pem

    # Force SSL/TLS redistribute: Redirect HTTP to HTTPS
    redirect scheme https if !{ ssl_fc }

    # Pass the client IP to the backend servers
    option forwardfor

    # Define Access Control Lists (ACLs)
    acl inside_subnet src 10.0.0.0/16
    acl it_acl hdr(host) -i it.company.com

    # Domain and Subnet based routing policies
    use_backend it_backend if it_acl inside_subnet
    use_backend intra_backend if inside_subnet
    
    # Fallback routing for all other traffic
    default_backend internet_backend
EOF
```

**Command Breakdown & Explanation:**

- `bind *:443 ssl crt /ca/server.pem`: Binds to port 443 and enables SSL termination using the combined PEM file.
- `if !{ ssl_fc }`: A built-in HAProxy fetch method meaning "if not SSL Front Channel" (i.e., if the connection arrived via plain HTTP).
- `src 10.0.0.0/16`: Matches the source IP of the incoming packet against the defined subnet.
- `hdr(host) -i it.company.com`: Matches the HTTP `Host` header exactly (case-insensitive due to `-i`).
- `use_backend`: Routes traffic sequentially. The first condition to match wins.

## 3. Backend Configuration and Load Balancing

The `backend` blocks define the pools of servers that will process the traffic. This section introduces load balancing algorithms and active HTTP health checks.

```Bash
# Append the backend configurations to /etc/haproxy/haproxy.cfg
cat << 'EOF' >> /etc/haproxy/haproxy.cfg

backend it_backend
    # Active health checks on the backend
    server host01 10.10.10.101:9000 check

backend intra_backend
    server host01 10.10.10.101:8081 check

backend internet_backend
    # Load balancing algorithm
    balance roundrobin
    
    # Layer 7 HTTP health checks instead of basic TCP pings
    option httpchk HEAD / HTTP/1.1\r\nHost:\ localhost
    
    # Server pool definitions
    server host01 10.10.10.101:8080 check
    server host02 10.10.10.102:8080 check
    server host03 10.10.10.103:8080 check
    server host04 10.10.10.104:8080 check
EOF
```

**Command Breakdown & Explanation:**

- `balance roundrobin`: Distributes incoming requests sequentially across all available servers in the pool. (Alternatives include `leastconn` or `source`).
- `check`: Instructs HAProxy to actively monitor the health of the backend server. If it fails, it is temporarily removed from the rotation.
- `option httpchk`: Upgrades the `check` mechanism from a basic TCP connection test to an actual HTTP request, ensuring the web server daemon is actively returning valid HTTP responses.

## 4. HAProxy Statistics Dashboard (Optional Feature)

HAProxy includes a built-in web dashboard to monitor the real-time status of frontends, backends, and server health states.

> [!WARNING]
> Always secure the statistics page with authentication to prevent unauthorized reconnaissance of your internal backend topology.

```Bash

# Append the statistics frontend to /etc/haproxy/haproxy.cfg
cat << 'EOF' >> /etc/haproxy/haproxy.cfg

frontend stats_page
    bind *:8404
    stats enable
    stats uri /haproxy?stats
    stats refresh 10s
    stats auth admin:WorldSkills2026!
EOF

# Restart the service to apply all configuration changes
systemctl restart haproxy
```

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the configuration file syntax, service status, and network listening ports on Debian 13 using standard administrative tools.

### 5.1 Verify HAProxy configuration syntax

**Command:** `haproxy -c -f /etc/haproxy/haproxy.cfg`

**What it checks and variables to look for:**

- **Output**: Must return `Configuration file is valid`. If there are structural errors, missing certificates, or ACL typos, it will output the exact line number causing the failure.

### 5.2 Verify HAProxy service status

**Command:** `systemctl status haproxy`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/haproxy.service; enabled)`

### 5.3 Verify network listening state on configured ports

**Command:** `ss -tulnp | grep haproxy`

**What it checks and variables to look for:**

- **HTTP/HTTPS**: Must display `LISTEN` on `*:80` and `*:443`.
- **Stats Page**: Must display `LISTEN` on `*:8404` (if configured).

<!-- Created by: Gergő Téringer, 2026 -->