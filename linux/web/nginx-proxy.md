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

## Proxy configuration

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

## Health check configuration
Create a bash script that does the check for you and then append your servers into a file, which will hold backend servers.

### `/etc/nginx/dns.sh`
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

Create a cronjob or a timer service to run this script time to time when you want to run it, and you're done, you have nginx proxy with health checks.