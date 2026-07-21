<!-- 
---
title: "Nginx as proxy (Free version)"
author: "Gergő Téringer"
---
 -->
# Nginx as proxy (Free version)

## Install packages

You need more things to achieve "streaming"

```shell
apt install nginx nginx-full
```

## `/etc/nginx/nginx.conf`

Add to the end outside of http { ... }. After this we will put any configuration under /etc/nginx/stream.d

```cfg
stream {
    include "stream.d/*conf"
}
```

## WEB

### Frontend

#### Routing logic

```shell
http {
    #...
    geo $inside_subnet {
        default        0;
        10.0.0.0/16    1;
    }

    map $inside_subnet $default_route {
        1       http://intra_backend;
        0       http://internet_backend;
    }

    map $inside_subnet $it_route {
        1       http://it_backend;
        0       http://internet_backend;
    }
    # ...
}
```

#### HTTP

```shell
frontend http-in
    bind *:80

    option forwardfor # Include X-Forwarded-For header

    acl inside_subnet src 10.0.0.0/16
    acl it_acl hdr(host) -i it.company.com

    use_backend it_backend if it_acl inside_subnet
    use_backend intra_backend if inside_subnet
    default_backend internet_backend
```

#### HTTPS

```shell
server {
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate     /ca/server.crt;
    ssl_certificate_key /ca/server.key;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        proxy_pass $default_route; 
    }
}

server {
    listen 443 ssl;
    server_name it.company.com;

    ssl_certificate     /ca/server.crt;
    ssl_certificate_key /ca/server.key;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        proxy_pass $it_route; 
    }
}
```

### Upstream

This upstream config doesn't includes healthchecks

```shell
upstream it_backend {
    server 10.10.10.101:9000;
}

upstream intra_backend {
    server 10.10.10.101:8081;
}

upstream internet_backend {
    server 10.10.10.101:8080;
    server 10.10.10.102:8080;
    server 10.10.10.103:8080;
    server 10.10.10.104:8080;
}
```

### SSL/TLS redistribute

Create a block which includes the following part to configure redistribution.

```shell
server {
    listen 80 default_server;
    server_name _; # Catch-all  incoming traffic

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$host$request_uri;
}
```

## UDP

### Proxy configuration

Create the directory and edit your files

```bash
mkdir -p /etc/nginx/stream.d/
touch /etc/nginx/stream.d/{dns-front.conf,dns-back.conf}
```

`/etc/nginx/stream.d/dns-front.conf`

```cfg
server {
    listen 53 udp;
    proxy_pass dns_servers;
    proxy_timeout 3s;
    proxy_responses 1; # Roundrobin, after one response to another backend server
}   
```

### Health check configuration

Create a bash script that does the check for you and then append your servers into a file, which will hold backend servers.

#### `/etc/nginx/dns.sh`

```bash
#!/bin/bash
SERVERS="10.10.10.101 10.10.10.102 10.10.10.103 10.10.10.104"
RECORD="www.company.com"
TMP="/tmp/nginx-dns-back"
TARGET="/etc/nginx/stream.d/dns-back.conf"
echo -e "# Generated at $(date)\nupstream dns_servers {\n" > $TMP
for SERVER in SERVERS
do
    LOOKUP=$(dig @$SERVER $RECORD A +short +time=2 +tries=1)
    if [ -n "$RESULT" ]; then
        echo "server $SERVER:53;" >> $TMP
    else
        echo "server $SERVER:53 down;" >> $TMP
    fi
done

if ! cmp -s "$TMP" "$TARGET"; then
    cp "$TMP" "$TARGET"

    if nginx -t >/dev/null 2>&1; then
        systemctl reload nginx
    fi
    
    rm "$TMP"
fi
```

Create a cronjob or a timer service to run this script, and you're done!

<!-- Created by: Gergő Téringer, 2026 -->