# Client (VPS node)

This directory contains everything that should run on each monitored VPS.

## Components

- `node_exporter` for host metrics (internal Docker network only; `uts: host` so Grafana “Nodename” matches the real hostname)
- `systemd_exporter` for systemd unit metrics (internal Docker network only)
- `nginx` (`metrics-proxy`) exposes **one** public port `9443` and routes:
  - `/metrics/node` → `node_exporter`
  - `/metrics/systemd` → `systemd_exporter`
- `promtail` for log shipping to central Loki

## Files

- `docker-compose.yml` - VPS services
- `nginx/nginx.conf` - reverse proxy for scrape endpoints
- `promtail/config.yml.template` - promtail config template (`MONITORING_SERVER_IP`, `LOG_HOST` from `.env`)
- `.env` - set `MONITORING_SERVER_IP` and `LOG_HOST` (not committed); copy from `.env.example`

## Quick start

1. Create `.env` and set:
   - `cp .env.example .env`
   - `MONITORING_SERVER_IP` — IP or DNS of the monitoring server where Loki listens on `3100`
   - `LOG_HOST` — stable name for this VPS in Loki (e.g. `web-prod-01` or FQDN)
2. Optionally edit `promtail/config.yml.template` (log paths only).
3. Start services:

```bash
docker compose up -d
```

4. Verify:
   - node metrics via proxy: `http://<this-vps>:9443/metrics/node`
   - systemd metrics via proxy: `http://<this-vps>:9443/metrics/systemd`
   - promtail health: `http://<this-vps>:9080/ready`

## Network requirements

- Incoming to VPS from monitoring server:
  - `9443/tcp` (Prometheus scrape via nginx; paths `/metrics/node` and `/metrics/systemd`)
- Outgoing from VPS to monitoring server:
  - `3100/tcp` (Loki push API)

## systemd_exporter notes

- Image `systemd_exporter` v0.7+ no longer supports `--path.procfs`; optional `/proc`-based collectors use the process namespace from `pid: host`.
- The container runs as root and uses `pid: host` plus read-only mounts to `systemd` and D-Bus so it can read unit state from the host (not from inside an isolated PID namespace).
- Tune which units are exported with exporter flags, for example:
  - `--systemd.collector.unit-include=yourapp\\.service|nginx\\.service`
  - `--systemd.collector.unit-exclude=.*\\.mount`
- If metrics are missing or the exporter cannot connect to D-Bus, confirm paths on your distro (`/run/dbus/system_bus_socket`, `/run/systemd`).

## Optional hardening

- Restrict who can reach port `9443` (firewall allowlist for the monitoring server IP or VPN subnet only).
- Add TLS or basic auth on nginx if you expose this port beyond a private network.
