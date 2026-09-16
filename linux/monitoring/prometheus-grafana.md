# Prometheus & Grafana

1. Install docker
```sh
apt install docker*
apt --fix-broken install
apt install docker*
```
2. Copy the files which has been given to you in a CD drive to your filesystem.
```sh
mount /dev/sr[n] /mnt
mkdir -p /docker/images && cd /docker/images
cp /mnt/* /docker/images/
```
3. Load in the images from those files
```sh
docker load -i prom...
docker load -i graf....
```
4. Create everything for Grafana and Prometheus on the filesystem and adjust permissions, and ownerships
```sh
mkdir -p /docker/{grafana,prometheus}/{ca,data}
touch /etc/prometheus/prometheus.yml
cp /ca/{ca.crt,subca.crt,chain.pem} /docker/grafana/ca
chown -R 472:472 /docker/grafana
chown -R nobody:nogroup /docker/prometheus
```
5. Create `/docker/images/compose.yml` file
```yml
monitor:
  services:
    prometheus:
      images: prom/prometheus
      network_mode: host
      restart: unless-stopped
      volumes:
        - /docker/prometheus/data:/prometheus
        # - /docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    grafana:
      images: grafana/grafana
      network_mode: host
      restart: unless-stopped
      volumes:
        - /docker/grafana/data:/var/lib/grafana
        - /docker/grafana/ca:/etc/grafana/certs
      environment:
        GF_SMTP_ENABLED: true
        GF_SMTP_HOST: mail.domain.com:465
        GF_SMTP_USER: grafana
        GF_SMTP_PASSWORD: Passw0rd!
        GF_SMTP_FROM_NAME: GRAFANA
        GF_SMTP_FROM_ADDRESS: grafana@domain.com
        SSL_CERTS_DIR: /etc/ssl/certs:/etc/grafana/certs
```
6. Start the containers
```sh
cd /docker/images
docker-compose up -d
```
7. Copy out from Prometheus the template and restart the container
> [!WARNING]
> Before compose, get rid of the comment in the comopose.yml file!!!!!
```sh
docker cp monitor-prometheus-1:/etc/prometheus/prometheus.yml /docker/prometheus/prometheus.yml
docker-compose up -d
```
8. Fill up the content of the `prometheus.yml` with your shit, and make it work after that. Here is the blackbox exporters config:
```yml
scrape_configs:
  - job_name: "blackbox"
    static_configs:
      targets:
        - https://public.domain.com
    metrics_path: /prob
    params:
      module: [http_2xx]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```
9. After this here are some scripts that will get you something:
```py
# CPU LOAD AVARAGE
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100 )

# Blackbox HTTP TEST
probe_success
```
