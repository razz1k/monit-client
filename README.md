# Client (VPS node)

This directory contains everything that should run on each monitored VPS.

## Components

- `node_exporter` for host metrics
- `promtail` for log shipping to central Loki

## Files

- `docker-compose.yml` - VPS services
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
   - node exporter: `http://<this-vps>:9100/metrics`
   - promtail health: `http://<this-vps>:9080/ready`

## Network requirements

- Incoming to VPS from monitoring server:
  - `9100/tcp` (node_exporter scrape)
- Outgoing from VPS to monitoring server:
  - `3100/tcp` (Loki push API)
