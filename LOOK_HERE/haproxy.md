# HAProxy source address-based routing

```
frontend f_http
    bind :::80
    acl is_cert hdr(host) -i cert.flychina.cn
    acl is_cdp hdr(host) -i cdp.flychina.cn
    acl is_www hdr(host) -i www.flychina.cn
    acl is_public hdr(host) -i public.flychina.cn

    http-request deny unless is_www or is_public or is_cert or is_cdp
    redirect scheme https if !{ ssl_fc } !is_cert !is_cdp

    use_backend b_cert if is_cert
    use_backend b_cdp if is_cdp

frontend f_https
    bind :::443 ssl cert /path/to/certchain_with_key.pem
    acl is_www hdr(host) -i www.flychina.cn
    acl is_public hdr(host) -i public.flychina.cn
    acl is_localaddr src 10.31.0.0/16 2001:db8:cedf::/48

    http-request deny unless is_www or is_public is_localaddr

    use_backend b_internal if is_www is_localaddr
    use_backend b_public if is_www !is_localaddr
    use_backend b_public if is_public is_localaddr

backend b_xxxx
    server web01 szx-web-1.flychina.cn:300x check
    server web02 szx-web-2.flychina.cn:300x check
```
