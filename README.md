# Client (VPS node)

This directory contains everything that should run on each monitored VPS.

## Components

- `node_exporter` for host metrics (internal Docker network only)
- `systemd_exporter` for systemd unit metrics (internal Docker network only)
- `nginx` (`metrics-proxy`) exposes **one** public port `9443` and routes:
  - `/metrics/node` → `node_exporter`
  - `/metrics/systemd` → `systemd_exporter`
- `promtail` for log shipping to central Loki

## Files

- `docker-compose.yml` - VPS services
- `nginx/nginx.conf` - reverse proxy for scrape endpoints
- `promtail/config.yml` - promtail client configuration

## Quick start

1. Adjust `promtail/config.yml`:
   - replace `MONITORING_SERVER_IP` with your central server IP
   - optionally update log paths/labels
2. Start services:

```bash
docker compose up -d
```

3. Verify:
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
