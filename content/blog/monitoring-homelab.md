---
title: monitoring-homelab
date: 2026-08-25
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# Monitoring using node_exporter, Prometheus, Grafana

## Goal

Get actual visibility into whether the lab is healthy, instead of only
finding out something's wrong when a manual SSH check happens to catch it.

## Why this exists

To see all the statics and information about each vm from a centralized dashboard 
instead of from the terminal via ssh.

## The model

- **Prometheus** — collects and stores numbers over time (a time-series
  database). CPU, memory, disk, whether a service is up.
- **Grafana** — visualizes whatever Prometheus has stored. Dashboards,
  graphs. Has no memory of its own; it queries Prometheus on demand.

Prometheus doesn't read machine internals directly — a small helper
program called an **exporter** runs on each monitored machine and exposes
metrics over plain HTTP. Prometheus **pulls** from each exporter on a
schedule ("scraping"), rather than machines pushing data to it.

```
node_exporter (on each VM) -> exposes metrics on :9100 ->
Prometheus (on router) -> scrapes every 15s -> stores it -> 
Grafana (on router) -> queries Prometheus -> renders dashboards
```

## Topology

- **`router`, `server`, `client`** — all run `node_exporter` on port 9100.
- **`router`** — additionally runs Prometheus (port 9090) and Grafana
  (port 3000).
- Prometheus scrapes the other two over the `lan` network using their
  `.homelab` DNS names (`router.homelab:9100`, `server.homelab:9100`,
  `client.homelab:9100`)

## Part 1 — node_exporter (on every VM)

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
```

**Dedicated service user**, following the principle of least privilege —
the exporter only needs to read metrics and serve HTTP, nothing more:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
```
`--no-create-home` — it's not a person, doesn't need a home directory.
`--shell /usr/sbin/nologin` — even if this account were ever compromised,
there's no way to get an interactive shell session out of it.

**Systemd service** (`/etc/systemd/system/node_exporter.service`):
```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
`After=network.target` — don't start before basic networking is up, since
this needs to bind a port. `Type=simple` — the `ExecStart` command *is*
the running service. `WantedBy=multi-user.target` — start automatically
once the system reaches normal boot state.

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
curl http://localhost:9100/metrics | head -20   # confirm real output
```

## Part 2 — Prometheus (router only)

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
tar xvf prometheus-2.54.1.linux-amd64.tar.gz
cd prometheus-2.54.1.linux-amd64
sudo mv prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv consoles console_libraries /etc/prometheus/
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus \
  /usr/local/bin/prometheus /usr/local/bin/promtool
```
The `chown` matters more here than for node_exporter — Prometheus
actually **writes** its time-series database to `/var/lib/prometheus`, not
just reads config.

**Scrape config** (`/etc/prometheus/prometheus.yml`):
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - 'router.homelab:9100'
          - 'server.homelab:9100'
          - 'client.homelab:9100'
```

Validate before running:
```bash
promtool check config /etc/prometheus/prometheus.yml
```

**Systemd service** (`/etc/systemd/system/prometheus.service`):
```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
```
`--web.listen-address=0.0.0.0:9090` — listens on all interfaces, not just
localhost, so it's reachable from `server`/`client` too if ever needed.

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
curl http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'
```
Expect `"health":"up"` three times.

## Part 3 — Grafana (router only)

```bash
sudo apt install -y wget gnupg
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Viewing the UIs from the host** — since `router`'s ports aren't
permanently forwarded, use an SSH tunnel on demand:
```bash
ssh -L 3000:localhost:3000 -L 9090:localhost:9090 -p 2224 vboxuser@127.0.0.1
```
Then open `http://localhost:3000` (Grafana) or `http://localhost:9090`
(Prometheus) in the host browser.

Or you can simpley enable port forwarding in virtual box.

**Connect Grafana to Prometheus:** Menu → Connections → Data sources →
Add data source → Prometheus → URL `http://localhost:9090` → Save & test.

**Import a ready-made dashboard** rather than building panels from
scratch: Menu → Dashboards → New → Import → dashboard ID **`1860`**
("Node Exporter Full") → select the Prometheus data source → Import.

## Part 4 — Basic PromQL

A few queries worth knowing, run from Prometheus's own UI
(`http://localhost:9090` → Graph):

```promql
up
```
`1` = target reachable, `0` = not. Generated automatically by Prometheus
for every scrape attempt, not something node_exporter reports itself.

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
CPU busy percentage per machine. `rate()` turns a cumulative counter
("total seconds idle ever") into a meaningful per-second trend over the
last 5 minutes.

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```
Memory used, as a percentage.

```promql
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024
```
Free disk space on `/`, in GB.

```promql
up == 0
```
Only the currently-unreachable targets — fast way to spot an outage.

## Ansible automation

- `tasks/monitoring/node_exporter.yml` — applied to `router,server,clients`
- `tasks/monitoring/prometheus.yml` + `tasks/monitoring/grafana.yml` —
  applied to `router` only
- `playbooks/monitoring.yml` — ties both together with handlers
  (`restart node_exporter`, `restart prometheus`)

Run:
```bash
ansible-playbook -i inventory.ini playbooks/monitoring.yml
```

Confirmed idempotent — a second run shows `changed=0` across all three
hosts (aside from a correctly-skipped conditional apt-cache-update task).

## Verification checklist

```bash
# on router
sudo systemctl status node_exporter prometheus grafana-server
curl -s http://localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'
```
All services active, all three targets `up`. In Grafana, the imported
dashboard should show live data for all three hosts via the dropdown

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
