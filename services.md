# PLEROMA — Service Reference

## Service Dependency Map

```
                    ┌─────────────┐
                    │  Homepage    │ (links to all services)
                    └──────┬──────┘
                           │ depends on all services being routable
                           │
     ┌─────────────────────┼─────────────────────┐
     │                     │                      │
┌────▼─────┐    ┌──────────▼──────────┐    ┌──────▼──────┐
│ Frigate   │    │ Monitoring Stack     │    │ DNS Stack   │
│           │    │                      │    │             │
│ ← Coral   │    │ Grafana ← Prometheus │    │ Pi-hole     │
│   USB     │    │ Uptime Kuma          │    │  ← Unbound  │
│           │    │ Watchtower           │    │             │
└───────────┘    └─────────────────────┘    └─────────────┘
     │                     │                      │
     │                     │                      │
     └─────────────────────┼──────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  WireGuard   │ (gateway for all remote access)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  SearXNG     │ (standalone, DNS via Pi-hole)
                    └─────────────┘
```

## Home Node Services

### Frigate

| Property | Value |
|----------|-------|
| Image | `ghcr.io/blakeblackshear/frigate:stable` |
| Port | 5000 (HTTP), 8554 (RTSP restream), 8555 (WebRTC) |
| Volumes | Config, recordings storage, Coral device passthrough |
| Dependencies | Google Coral USB (`/dev/bus/usb`), recording storage mounted |
| Network | Needs access to camera VLAN (RTSP streams) |
| Resource notes | Primary CPU consumer. Hardware decode via Intel Quick Sync (VAAPI). |
| Config path | `home/configs/frigate/` |

**Key configuration decisions:**
- Detection runs on Coral USB via sub stream (720p)
- Recording uses main stream (4K/1080p)
- Retention managed by Frigate config (disk percentage threshold)
- Zones and object filters configured per camera
- MQTT optional — only needed if integrating with Home Assistant (not in current scope)

### WireGuard

| Property | Value |
|----------|-------|
| Image | `linuxserver/wireguard` or native kernel module |
| Port | 51820 (UDP) — **only port exposed to internet** |
| Volumes | Config, peer configs |
| Dependencies | Kernel WireGuard module (included in modern kernels) |
| Network | Requires router port forward: external 51820/UDP → Home Node |
| Config path | `home/configs/wireguard/` |

**Key configuration decisions:**
- Server mode on Home Node
- Peer configs generated for each client device
- AllowedIPs determines split vs full tunnel
- PostUp/PostDown rules for firewall and routing
- Persistent keepalive for NAT traversal
- Separate peer for VPS-to-Home tunnel

### Pi-hole

| Property | Value |
|----------|-------|
| Image | `pihole/pihole:latest` |
| Port | 8080 (HTTP admin), 53 (DNS — internal only) |
| Volumes | Config, custom blocklists, local DNS records |
| Dependencies | Unbound (upstream DNS at 127.0.0.1:5335 or container IP) |
| Network | DNS traffic from WireGuard clients and local containers |
| Config path | `home/configs/pihole/` |

**Key configuration decisions:**
- Upstream DNS set to Unbound container (not external resolvers)
- Custom blocklists curated for privacy (not just ads)
- Local DNS records for service names (e.g., `frigate.home` → container IP)
- DHCP disabled (router handles DHCP)
- Rate limiting configured appropriately for number of clients

### Unbound

| Property | Value |
|----------|-------|
| Image | `mvance/unbound:latest` or `klutchell/unbound` |
| Port | 5335 (DNS — Pi-hole upstream only) |
| Volumes | Config |
| Dependencies | None (root hints built-in) |
| Network | Only Pi-hole should query Unbound |
| Config path | `home/configs/unbound/` |

**Key configuration decisions:**
- Recursive resolver — queries root DNS servers directly
- DNSSEC validation enabled
- No forwarding to any upstream provider
- Aggressive NSEC caching for performance
- Access control: only Pi-hole container allowed

### SearXNG

| Property | Value |
|----------|-------|
| Image | `searxng/searxng:latest` |
| Port | 8888 (HTTP) |
| Volumes | Config, settings |
| Dependencies | DNS resolution (via Pi-hole/Unbound) |
| Network | Needs outbound internet for search engine queries |
| Config path | `home/configs/searxng/` |

**Key configuration decisions:**
- Search engines selected for privacy and quality
- Image proxy enabled (routes image results through SearXNG)
- Rate limiting configured
- Auto-complete enabled
- Default theme and preferences pre-configured
- Accessible via WireGuard only

### Grafana

| Property | Value |
|----------|-------|
| Image | `grafana/grafana:latest` |
| Port | 3000 (HTTP) |
| Volumes | Dashboards, data sources, database |
| Dependencies | Prometheus (data source) |
| Network | Internal only |
| Config path | `home/configs/grafana/` |

**Key configuration decisions:**
- Pre-provisioned data source (Prometheus)
- Pre-built dashboards: system resources, Frigate metrics, Pi-hole stats, WireGuard peers, container health
- Anonymous access or single-user auth (behind WireGuard, not internet-facing)
- Alerting rules for critical conditions (disk full, service down, high CPU)

### Prometheus

| Property | Value |
|----------|-------|
| Image | `prom/prometheus:latest` |
| Port | 9090 (HTTP) |
| Volumes | Config, TSDB data |
| Dependencies | Scrape targets (all other services with metrics endpoints) |
| Network | Internal only |
| Config path | `home/configs/prometheus/` |

**Key configuration decisions:**
- Scrape interval: 15s default, 60s for less critical targets
- Retention: 30 days (adjustable based on disk)
- Scrape targets: node_exporter (system), Frigate, Pi-hole, WireGuard, Docker containers
- cAdvisor or Docker metrics for container-level monitoring
- node_exporter for system-level metrics (CPU, RAM, disk, network)

### Portainer

| Property | Value |
|----------|-------|
| Image | `portainer/portainer-ce:latest` |
| Port | 9443 (HTTPS) |
| Volumes | Docker socket (read-only recommended), Portainer data |
| Dependencies | Docker socket |
| Network | Internal only |
| Config path | `home/configs/portainer/` |

**Key configuration decisions:**
- Community Edition (free)
- Docker socket mounted read-only if possible
- Single admin user
- Used for container management, log viewing, quick restarts
- Not for deployment (Compose handles that)

### Watchtower

| Property | Value |
|----------|-------|
| Image | `containrrr/watchtower:latest` |
| Port | 8085 (metrics, optional) |
| Volumes | Docker socket (read-only) |
| Dependencies | Docker socket |
| Network | Needs outbound internet to check registries |
| Config path | `home/configs/watchtower/` |

**Key configuration decisions:**
- **MONITOR ONLY — no auto-updates** (`WATCHTOWER_MONITOR_ONLY=true`)
- Notification on available updates (Telegram, email, or Discord)
- Schedule: daily check
- Updates are applied manually after testing on operator's own stack
- This is a deliberate stability decision for client deployments

### Uptime Kuma

| Property | Value |
|----------|-------|
| Image | `louislam/uptime-kuma:latest` |
| Port | 3001 (HTTP) |
| Volumes | Data directory |
| Dependencies | None |
| Network | Internal (checks other containers via Docker network) |
| Config path | `home/configs/uptime-kuma/` |

**Key configuration decisions:**
- HTTP/HTTPS checks for all web UIs (Frigate, Pi-hole, SearXNG, Grafana, etc.)
- TCP checks for DNS (Pi-hole port 53), WireGuard
- Ping checks for cameras
- Notification channels: Telegram and/or email
- Check interval: 60s for services, 300s for cameras
- Status page: optional, can be served via Homepage

### Homepage

| Property | Value |
|----------|-------|
| Image | `ghcr.io/gethomepage/homepage:latest` |
| Port | 3010 (HTTP) |
| Volumes | Config |
| Dependencies | All other services (for widget integrations) |
| Network | Internal only |
| Config path | `home/configs/homepage/` |

**Key configuration decisions:**
- Service widgets showing live status (Frigate, Pi-hole stats, system resources)
- Bookmarks to all service UIs
- Clean layout grouped by function (Surveillance, Network, Monitoring, Search)
- Docker integration for container status
- This becomes the single entry point / home screen bookmark

---

## Foreign VPS Services

The VPS runs a minimal stack: WireGuard + Pi-hole + Unbound.

Configuration mirrors the Home Node equivalents with these differences:

| Service | VPS Difference |
|---------|---------------|
| WireGuard | Server mode. Peers: client devices + Home Node tunnel. NAT masquerade for internet exit. |
| Pi-hole | Standalone instance. Separate blocklists (can be synced). No local DNS records for home services. |
| Unbound | Identical recursive resolver configuration. |

VPS config files live in `vps/configs/`.

---

## Docker Network Strategy

```
Networks:
  frontend:     # Services with web UIs (Homepage, Grafana, Frigate, etc.)
  backend:      # Internal service communication (Prometheus → targets)
  dns:          # Pi-hole ↔ Unbound
  cameras:      # Frigate ↔ camera VLAN (macvlan or host network segment)
```

Services are attached to only the networks they need. This limits lateral movement if a container is compromised.

---

## Startup Order

1. **Unbound** — DNS resolver must be available first
2. **Pi-hole** — Depends on Unbound for upstream
3. **WireGuard** — Depends on networking being stable
4. **Frigate** — Can start independently but benefits from DNS
5. **SearXNG** — Needs outbound DNS
6. **Prometheus** — Needs scrape targets to be available
7. **Grafana** — Needs Prometheus data source
8. **Uptime Kuma** — Needs targets to monitor
9. **Watchtower** — Needs Docker socket
10. **Portainer** — Needs Docker socket
11. **Homepage** — Needs all services running for widget data

Docker Compose `depends_on` with health checks will enforce this ordering.
