<!-- 
---
title: "HaProxy"
author: "Gergő Téringer"
---
-->
# HaProxy

## WEB

### Frontend

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
frontend http-in
    bind *:443 ssl /ca/server.pem # Use certificate chain including the private key and the certificate

    option forwardfor # Include X-Forwarded-For header

    acl inside_subnet src 10.0.0.0/16
    acl it_acl hdr(host) -i it.company.com

    use_backend it_backend if it_acl inside_subnet
    use_backend intra_backend if inside_subnet
    default_backend internet_backend
```

### Backend

The `check` command issues healthchecks to backend servers.

```shell
backend it_backend
    server  host01   10.10.10.101:9000 check

backend intra_backend
    server  host01   10.10.10.101:8081 check

backend internet_backend
    server  host01   10.10.10.101:8080 check
    server  host02   10.10.10.102:8080 check
    server  host03   10.10.10.103:8080 check
    server  host04   10.10.10.104:8080 check
```

### SSL/TLS redistribute

In the frontend where you want to implement SSL redistribute (so where your web traffic comes in) implement this line:

```shell
    redirect scheme https if !{ ssl_fc }
```

<!-- Created by: Gergő Téringer, 2026 -->