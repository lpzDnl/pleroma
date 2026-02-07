# PLEROMA — Maintenance & Operations

## Update Strategy

### Philosophy

> **Stability over freshness.** This is a client deployment. Unattended updates that break services erode trust and create emergency support calls. All updates are tested first, applied deliberately.

### Update Pipeline

```
1. Watchtower detects new image   →  Notification sent to operator
2. Operator reviews changelog      →  Breaking changes? Config migration needed?
3. Operator tests on own stack     →  Run updated image in personal environment
4. Operator validates stability    →  24-48 hour soak test
5. Schedule client update          →  Communicate maintenance window if needed
6. Apply update to client stack    →  docker compose pull && docker compose up -d
7. Verify post-update              →  Check Uptime Kuma, Grafana, Frigate recordings
```

### Update Cadence

| Category | Frequency | Examples |
|----------|-----------|---------|
| Security patches | ASAP (within 48h of operator validation) | CVEs in base images, WireGuard, exposed services |
| Minor updates | Monthly batch | Pi-hole blocklist updates, Grafana minor versions |
| Major updates | Quarterly or as needed | Frigate major versions, OS kernel updates |
| Base OS | Quarterly | Ubuntu LTS point releases, kernel updates |

### Watchtower Configuration

```
WATCHTOWER_MONITOR_ONLY=true
WATCHTOWER_NOTIFICATIONS=true
WATCHTOWER_NOTIFICATION_URL=telegram://token@telegram?chats=chatid
WATCHTOWER_SCHEDULE=0 0 4 * * *   # Check daily at 4 AM
```

Watchtower checks for updates but **never applies them automatically**. Notifications go to the operator (Daniel), not the client.

---

## Backup Strategy

### What to Back Up

| Data | Location | Frequency | Method |
|------|----------|-----------|--------|
| Docker Compose files | `/opt/pleroma/` | On every change | Git (this repo) |
| Service configs | `/opt/pleroma/home/configs/` | On every change | Git (this repo) |
| Pi-hole config + blocklists | Docker volume | Weekly | Volume export script |
| Grafana dashboards | Docker volume | Weekly | Grafana provisioning (config-as-code in repo) |
| Prometheus TSDB | Docker volume | Not backed up | Metrics are ephemeral; 30-day retention, rebuildable |
| Frigate recordings | Recording drive | Not backed up | Too large; retention handles lifecycle |
| Frigate config + database | Docker volume | Weekly | Volume export script |
| WireGuard configs + keys | Docker volume | On every change | Encrypted backup (keys are sensitive) |
| Uptime Kuma database | Docker volume | Weekly | Volume export script |

### What NOT to Back Up

- Frigate video recordings (multi-TB, retention-managed)
- Prometheus time-series data (ephemeral by design)
- Container images (re-pulled from registries)
- Logs (ephemeral, monitored in real-time)

### Backup Script (Future)

A `scripts/backup.sh` script will:
1. Export critical Docker volumes to compressed archives
2. Encrypt sensitive data (WireGuard keys)
3. Store locally and optionally push to VPS or off-site
4. Run on a weekly cron schedule
5. Notify operator on success/failure

---

## Monitoring & Alerting

### Alert Conditions

| Condition | Severity | Detection | Action |
|-----------|----------|-----------|--------|
| Disk usage > 85% | Warning | Prometheus + Grafana | Review Frigate retention, clean up |
| Disk usage > 95% | Critical | Prometheus + Grafana | Immediate intervention |
| Service down > 5 min | Critical | Uptime Kuma | Restart container, investigate |
| Camera offline | Warning | Frigate + Uptime Kuma | Check physical connection, power |
| WireGuard tunnel down | Critical | Uptime Kuma | Check VPS, check Home Node |
| High CPU > 90% sustained | Warning | Prometheus | Investigate Frigate load, container runaway |
| High RAM > 90% | Warning | Prometheus | Check for memory leaks, OOM risk |
| Container restart loop | Critical | Portainer / Docker events | Check logs, rollback if needed |
| SSL/TLS cert expiry | Warning | Uptime Kuma | Renew (if applicable) |
| Coral USB disconnected | Critical | Frigate logs / metrics | Reseat USB, check dmesg |

### Notification Channels

| Channel | Use Case |
|---------|----------|
| Telegram | Primary — operator receives all alerts |
| Email | Secondary — critical alerts only |

Client does not receive raw alerts. Operator triages and communicates as needed.

---

## Troubleshooting Runbook

### Service Won't Start

```bash
# Check container logs
docker compose logs <service-name> --tail 50

# Check if port is already in use
ss -tlnp | grep <port>

# Check Docker events
docker events --since 10m

# Restart single service
docker compose restart <service-name>

# Nuclear option — recreate
docker compose up -d --force-recreate <service-name>
```

### Frigate Not Recording

1. Check Frigate UI → are camera streams visible?
2. Check recording storage: `df -h /path/to/recordings`
3. Check Frigate logs: `docker compose logs frigate --tail 100`
4. Verify Coral is detected: `lsusb | grep Google`
5. Check camera RTSP: `ffprobe rtsp://camera-ip:554/path`

### DNS Not Resolving

1. Check Pi-hole container: `docker compose logs pihole`
2. Check Unbound: `docker compose logs unbound`
3. Test Unbound directly: `dig @127.0.0.1 -p 5335 google.com`
4. Test Pi-hole: `dig @127.0.0.1 -p 53 google.com`
5. Check WireGuard client DNS setting points to Pi-hole IP

### WireGuard Can't Connect

1. Check WireGuard is running: `docker compose logs wireguard`
2. Verify port forward on router: external 51820/UDP → Home Node IP
3. Check firewall: `iptables -L -n | grep 51820`
4. Check peer config matches server config (public keys, endpoints)
5. Test from client: `wg show` for handshake status

### Camera Offline

1. Check PoE switch — is the port active/powered?
2. Ping camera IP from Home Node
3. Check physical cable connection
4. Try camera web UI directly (most Reolink cameras have web admin)
5. Power cycle via PoE switch port disable/enable

---

## Scheduled Maintenance Calendar

| Task | Frequency | Window |
|------|-----------|--------|
| Review Watchtower notifications | Weekly | Operator's discretion |
| Check disk usage and Frigate retention | Bi-weekly | N/A |
| Review Pi-hole query logs and blocklists | Monthly | N/A |
| Apply tested container updates | Monthly | Low-usage hours |
| Review Grafana dashboards for anomalies | Monthly | N/A |
| Full backup verification (restore test) | Quarterly | Planned maintenance window |
| OS updates and kernel patches | Quarterly | Planned maintenance window with reboot |
| Review WireGuard peer list | Quarterly | N/A |
| Hardware check (thermals, disk health) | Quarterly | N/A |
| Annual security audit | Annually | Planned |

---

## Incident Response

### Severity Levels

| Level | Definition | Response Time |
|-------|-----------|---------------|
| P1 — Critical | Service outage affecting security (cameras down, VPN down) | < 4 hours |
| P2 — High | Service degraded (slow performance, single camera offline) | < 24 hours |
| P3 — Medium | Non-critical service issue (Grafana down, SearXNG slow) | < 72 hours |
| P4 — Low | Cosmetic or minor (Homepage widget broken, blocklist stale) | Next scheduled maintenance |

### Incident Template

```
Date:
Severity:
Service(s) affected:
Symptoms:
Root cause:
Resolution:
Time to detect:
Time to resolve:
Prevention measures:
```

Incidents are logged in `docs/incidents/` for pattern analysis and improvement.
