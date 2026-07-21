<!-- 
---
title: "Nginx"
author: "Gergő Téringer"
---
-->
# Nginx

## HTTP -> HTTPS redirect setup

```bash
server {
    listen 80;
    listen [::]:80;

    server_name _; # This will catch all hosts! Edit the to the dns name to the virtualhost!

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    ssl_certificate /ca/server.crt
    ssl_certificate_key /ca/server.key

    server_name _; # This will catch all hosts! Edit the to the dns name to the virtualhost!

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

<!-- Created by: Gergő Téringer, 2026 -->