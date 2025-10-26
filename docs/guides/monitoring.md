# Monitoring — Prometheus + Grafana (Docker Compose)

> Lightweight stack to monitor homelab hosts, containers, and network devices.

## Overview

- **Prometheus**: metrics TSDB
- **Node Exporter**: host metrics
- **cAdvisor**: container metrics
- **Grafana**: dashboards + alerts
- Optional: **Blackbox Exporter** to probe HTTP/TCP/ICMP

## Directory layout

```
monitoring/
├─ docker-compose.yml
├─ prometheus/
│  ├─ prometheus.yml
│  └─ file_sd/
│     └─ targets.json
└─ grafana/
   └─ provisioning/
      ├─ datasources/datasource.yml
      └─ dashboards/
```

## 1) docker-compose.yml

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/file_sd:/etc/prometheus/file_sd:ro
      - prom_data:/prometheus
    ports: [ "9090:9090" ]
    restart: unless-stopped

  nodeexporter:
    image: prom/node-exporter
    pid: host
    network_mode: host
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports: [ "8080:8080" ]
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=changeme
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    ports: [ "3000:3000" ]
    restart: unless-stopped

volumes:
  prom_data:
  grafana_data:
```

## 2) Prometheus config

`prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node'
    file_sd_configs:
      - files: ['/etc/prometheus/file_sd/targets.json']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

`prometheus/file_sd/targets.json` (add your hosts):

```json
[
  { "targets": ["192.168.30.11:9100"], "labels": { "env": "lab", "role": "pve01" } },
  { "targets": ["192.168.30.12:9100"], "labels": { "env": "lab", "role": "nas" } }
]
```

Install **node_exporter** on baremetal hosts:
```bash
# Debian/Ubuntu example (systemd service)
useradd --no-create-home --shell /usr/sbin/nologin node_exporter || true
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xzf node_exporter-*.tar.gz
cp node_exporter-*/node_exporter /usr/local/bin/
cat >/etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Node Exporter
After=network-online.target
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now node_exporter
```

## 3) Grafana provisioning

`grafana/provisioning/datasources/datasource.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

You can import community dashboards:
- Node Exporter Full (ID **1860**)
- Prometheus 2.0 Stats (ID **3662**)
- cAdvisor (ID **14282**)

## 4) Bring it up

```bash
docker compose up -d
# Prometheus: http://<host>:9090
# Grafana:    http://<host>:3000  (admin / changeme)
```

## 5) Alerts (quick start)

Add to `prometheus/prometheus.yml`:
```yaml
rule_files:
  - alert.rules.yml
```
Create `prometheus/alert.rules.yml`:
```yaml
groups:
- name: basic
  rules:
  - alert: HostDown
    expr: up{job="node"} == 0
    for: 2m
    labels: { severity: critical }
    annotations:
      summary: "Host down"
      description: "No metrics from {{ $labels.instance }}"
```

Wire Alertmanager later to route to email/Telegram/Slack.

## 6) Backup & update

- Backup Grafana dashboards: `grafana_data` volume.
- Backup Prometheus data if needed (TSDB can be rebuilt).
- Update images periodically: `docker compose pull && docker compose up -d`.

## Troubleshooting

=== "No targets in Prometheus"
    - Check `targets.json` IP:port and firewall

=== "Grafana can't connect"
    - Confirm datasource URL `http://prometheus:9090`
