# Role: monitoring

Deploys a complete monitoring stack using **Prometheus** and **Node Exporter**.  
Node Exporter runs on every host and exposes system metrics. Prometheus runs on `monitor01` and scrapes all hosts every 15 seconds.

---

## Architecture

```
webserver01 :9100 (Node Exporter) ←─┐
dbserver01  :9100 (Node Exporter) ←─┤── Prometheus :9090 (monitor01)
monitor01   :9100 (Node Exporter) ←─┘
```

---

## What it does

### On ALL hosts
| Task | Description |
|------|-------------|
| Create node_exporter user | Dedicated system user, no shell, no home |
| Download node_exporter | Binary from GitHub releases |
| Install node_exporter | Binary to `/usr/local/bin/node_exporter` |
| Deploy systemd service | Manages node_exporter as a system service |
| Enable node_exporter | Starts on boot, runs continuously |
| Cleanup temp files | Removes downloaded archives |

### On monitor01 only
| Task | Description |
|------|-------------|
| Create prometheus user | Dedicated system user, no shell, no home |
| Create directories | `/etc/prometheus` and `/var/lib/prometheus` |
| Download prometheus | Binary from GitHub releases |
| Install prometheus | Binaries to `/usr/local/bin/` |
| Deploy prometheus.yml | Scrape config generated from Ansible inventory |
| Deploy systemd service | Manages prometheus as a system service |
| Enable prometheus | Starts on boot, runs continuously |
| Cleanup temp files | Removes downloaded archives |

---

## Requirements

- Port 9100 open in UFW on all hosts (defined in group_vars)

---

## Variables

Defined in `roles/monitoring/defaults/main.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `node_exporter_version` | `1.8.2` | Node Exporter version to install |
| `prometheus_version` | `2.53.0` | Prometheus version to install |
| `node_exporter_port` | `9100` | Port Node Exporter listens on |
| `prometheus_port` | `9090` | Port Prometheus listens on |
| `prometheus_scrape_interval` | `15s` | How often Prometheus scrapes targets |
| `prometheus_evaluation_interval` | `15s` | How often Prometheus evaluates rules |
| `prometheus_retention_time` | `15d` | How long to keep metrics data |

---

## Templates

| Template | Destination | Description |
|----------|------------|-------------|
| `node_exporter.service.j2` | `/etc/systemd/system/node_exporter.service` | Node Exporter systemd service |
| `prometheus.yml.j2` | `/etc/prometheus/prometheus.yml` | Prometheus scrape configuration |
| `prometheus.service.j2` | `/etc/systemd/system/prometheus.service` | Prometheus systemd service |

---

## Handlers

| Handler | Triggered by | Action |
|---------|-------------|--------|
| Restart node_exporter | Service file changes | `systemctl restart node_exporter` |
| Restart prometheus | Config or service file changes | `systemctl restart prometheus` |

---

## Firewall notes

Port 9100 must be open on all hosts. This is configured in `inventories/lab/group_vars/`:

```yaml
# webservers.yml and dbservers.yml
hardening_allowed_ports:
  - { port: "9100", proto: "tcp" }

# monitoring.yml
hardening_allowed_ports:
  - { port: "9090", proto: "tcp" }
  - { port: "9100", proto: "tcp" }
```

---

## Usage

```bash
ansible-playbook playbooks/monitoring.yml
```

---

## How to verify

### Node Exporter — all hosts
```bash
# Check service status inside VM
vagrant ssh webserver01
sudo systemctl status node_exporter

# Check metrics endpoint
curl http://localhost:9100/metrics | head -5
exit

# From host — all 3 must respond
curl -s http://192.168.56.10:9100/metrics | head -3
curl -s http://192.168.56.11:9100/metrics | head -3
curl -s http://192.168.56.12:9100/metrics | head -3
```

Expected output:
```
# HELP go_gc_duration_seconds ...
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} ...
```

### Prometheus — monitor01
```bash
vagrant ssh monitor01

# Service running?
sudo systemctl status prometheus

# Config valid?
promtool check config /etc/prometheus/prometheus.yml
# SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax

exit
```

### All targets UP
```bash
curl -s http://192.168.56.12:9090/api/v1/targets \
  | python3 -m json.tool | grep -E "scrapeUrl|health"
```

Expected — all 3 targets must show `"health": "up"`:
```json
"scrapeUrl": "http://192.168.56.10:9100/metrics",
"health": "up",
"scrapeUrl": "http://192.168.56.11:9100/metrics",
"health": "up",
"scrapeUrl": "http://192.168.56.12:9100/metrics",
"health": "up",
```

---

## Prometheus Web UI

Open your browser at `http://192.168.56.12:9090`

### Status → Targets
Shows all scrape targets with their current state (UP/DOWN) and last scrape time.

### Graph — useful queries

**CPU usage % per host:**
```
100 - (avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

**RAM available in MB:**
```
node_memory_MemAvailable_bytes / 1024 / 1024
```

**RAM used %:**
```
100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)
```

**Disk used %:**
```
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)
```

**Load average:**
```
node_load1
```

> Note: queries using `rate()` require at least 1-2 minutes of data after deployment.

---

## Technical notes

**System users** — Both `node_exporter` and `prometheus` run as dedicated system users with no shell (`/sbin/nologin`) and no home directory. If a process is compromised, the attacker has no usable shell.

**`when: "'monitoring' in group_names"`** — Prometheus tasks use this condition to run only on hosts in the `monitoring` inventory group. Node Exporter tasks have no condition and run on all hosts. One role handles both components cleanly.

**`promtool check config`** — The prometheus.yml template uses `validate` to test the configuration before applying it. If the config has errors, Prometheus is never restarted with a broken config.

**`--web.enable-lifecycle`** — Enables config reload without restart: `curl -X POST http://localhost:9090/-/reload`

**Idempotency** — The `stat` module checks if binaries already exist before downloading. Re-running the playbook skips the download if already installed.

**Inventory-driven targets** — The Jinja2 loop in `prometheus.yml.j2` generates scrape targets directly from the Ansible inventory. Adding a new host to the inventory and re-running the playbook automatically updates Prometheus configuration.
