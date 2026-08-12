# NGINX reverse proxy with TLS termination

Install nginx.

```bash
apt install nginx
```

No extra module is needed.

No need to turn on any module.

Create a config file:

`/etc/nginx/sites-available/grafana.conf`

```conf
server {
    listen 80;
    listen [::]:80;
    server_name grafana.flychina.cn;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name grafana.flychina.cn;

    ssl_certificate /path/to/certificate_chain.pem;
    ssl_certificate_key /path/to/key;

    location / {
        proxy_pass http://127.0.0.1:3000; # Or wherever the backend is
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Create a symlink for this file in `/etc/nginx/sites-enabled`:

```bash
ln -s /etc/nginx/sites-available/grafana.conf /etc/nginx/sites-enabled/grafana.conf
```

**Delete any other symlinks in this folder!!**
