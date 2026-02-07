# PLEROMA — Architecture

## Network Topology

```
                           ┌─────────────────────────────────────────┐
                           │           INTERNET                      │
                           └──────────┬──────────────────┬───────────┘
                                      │                  │
                                      │                  │
                           ┌──────────▼──────────┐  ┌────▼──────────────────┐
                           │   CLIENT DEVICES     │  │   FOREIGN VPS         │
                           │                      │  │   (HostHatch)         │
                           │  Phone / Laptop /    │  │                       │
                           │  Tablet              │  │  WireGuard Server     │
                           │                      │  │  Pi-hole + Unbound    │
                           │  WireGuard Profiles:  │  │                       │
                           │  • Home tunnel       │  │  Encrypted exit node  │
                           │  • Foreign tunnel    │  │  DNS filtering abroad │
                           └──────────┬───────────┘  └────▲──────────────────┘
                                      │                   │
                              WireGuard tunnels           │ WireGuard tunnel
                                      │                   │ (Home ↔ VPS)
                                      │                   │
                           ┌──────────▼───────────────────┴──────────┐
                           │              HOME ROUTER                 │
                           │         (ISP / existing network)         │
                           └──────────┬──────────────────────────────┘
                                      │
                                      │ LAN
                                      │
            ┌─────────────────────────┴──────────────────────────┐
            │                                                     │
   ┌────────▼────────────────────────────┐    ┌──────────────────▼───────────┐
   │         HOME NODE (Mini PC)          │    │      PoE SWITCH              │
   │                                      │    │      (8-port managed)        │
   │  ┌─────────────────────────────┐     │    │                              │
   │  │ Docker Engine               │     │    │  VLAN: Camera Network        │
   │  │                             │     │    │  (isolated from LAN)         │
   │  │  Frigate ─── Coral USB      │     │    └──┬──┬──┬──┬──┬──┬──────────┘
   │  │  WireGuard Server           │     │       │  │  │  │  │  │
   │  │  Pi-hole + Unbound          │     │       │  │  │  │  │  │
   │  │  SearXNG                    │     │       ▼  ▼  ▼  ▼  ▼  ▼
   │  │  Grafana + Prometheus       │     │      ┌──────────────────┐
   │  │  Portainer                  │     │      │  PoE CAMERAS     │
   │  │  Watchtower (notify-only)   │     │      │  4x 4K outdoor   │
   │  │  Uptime Kuma                │     │      │  2x 1080p indoor │
   │  │  Homepage                   │     │      └──────────────────┘
   │  │                             │     │
   │  └─────────────────────────────┘     │
   │                                      │
   │  Storage:                            │
   │  • NVMe 1: OS + Docker volumes       │
   │  • NVMe 2 / HDD: Frigate recordings  │
   │                                      │
   └──────────────────────────────────────┘
```

## Service Interconnections

### DNS Resolution Chain

```
Client Device
  → WireGuard tunnel (home or foreign)
    → Pi-hole (ad/tracker filtering)
      → Unbound (recursive DNS resolution)
        → Root DNS servers (no upstream forwarder)
```

Both the Home Node and Foreign VPS run independent Pi-hole + Unbound instances. The client's active WireGuard profile determines which Pi-hole handles their DNS:

- **Home tunnel active** → Home Pi-hole → Home Unbound → root servers
- **Foreign tunnel active** → VPS Pi-hole → VPS Unbound → root servers

### Camera Pipeline

```
PoE Cameras (RTSP streams)
  → PoE Switch (camera VLAN, isolated)
    → Frigate
      ├── Main stream (4K/1080p) → Recording storage
      ├── Sub stream (720p) → Coral USB → AI detection
      ├── Events → Prometheus metrics
      └── Snapshots/Clips → Event storage
```

Cameras exist on an isolated VLAN. Only the Home Node (Frigate) can access camera streams. Cameras have no internet access.

### Monitoring Data Flow

```
All Docker containers
  ├── Prometheus (scrapes metrics endpoints)
  │     └── Grafana (visualization + dashboards)
  │
  ├── Uptime Kuma (HTTP/TCP health checks)
  │     └── Notifications (Telegram / Email / Discord)
  │
  └── Watchtower (checks for container image updates)
        └── Notifications (notify-only, no auto-update)
```

### VPN Topology

```
┌─────────────┐     WireGuard      ┌─────────────┐
│  Home Node   │◄──────────────────►│ Foreign VPS  │
│  10.0.1.1    │    persistent      │  10.0.1.2    │
└──────┬───────┘    tunnel          └──────────────┘
       │
       │  WireGuard
       │  client tunnels
       │
  ┌────▼────┐
  │ Clients  │
  │ 10.0.1.x │
  └──────────┘
```

Client devices receive WireGuard profiles for both tunnels:
- **Home profile**: Routes traffic through Home Node. Access to all home services (Frigate, SearXNG, Homepage, etc.) + internet exits from home IP.
- **Foreign profile**: Routes traffic through VPS. Internet exits from foreign IP. DNS filtered by VPS Pi-hole.

## Port Map (Internal — Home Node)

| Port | Service | Protocol | Notes |
|------|---------|----------|-------|
| 51820 | WireGuard | UDP | Only port exposed to internet (via router port forward) |
| 5000 | Frigate | HTTP | Web UI |
| 1883 | Frigate MQTT | TCP | Internal only (if MQTT broker added) |
| 8080 | Pi-hole | HTTP | Admin UI |
| 5335 | Unbound | TCP/UDP | DNS, Pi-hole upstream only |
| 8888 | SearXNG | HTTP | Search UI |
| 3000 | Grafana | HTTP | Dashboards |
| 9090 | Prometheus | HTTP | Metrics UI |
| 9443 | Portainer | HTTPS | Docker management |
| 3001 | Uptime Kuma | HTTP | Health monitoring |
| 3010 | Homepage | HTTP | Service dashboard |
| 8085 | Watchtower | HTTP | Metrics endpoint (if enabled) |

> **Note**: All service ports are internal only — accessible via WireGuard tunnel, never exposed to the internet. The only port forwarded on the router is WireGuard (51820/UDP).

## Port Map (Foreign VPS)

| Port | Service | Protocol | Notes |
|------|---------|----------|-------|
| 51820 | WireGuard | UDP | Exposed to internet |
| 8080 | Pi-hole | HTTP | Admin UI (via WireGuard only) |
| 5335 | Unbound | TCP/UDP | DNS, Pi-hole upstream only |

## Security Boundaries

1. **Camera VLAN** — Cameras isolated on their own VLAN. No internet access. Only the Home Node can reach camera RTSP streams.
2. **WireGuard-only access** — All services accessible only through WireGuard tunnels. Zero services exposed to the internet except WireGuard itself.
3. **DNS filtering** — All DNS queries pass through Pi-hole regardless of which VPN tunnel is active.
4. **Recursive DNS** — Unbound resolves directly against root servers. No reliance on upstream DNS providers (Google, Cloudflare, etc.).
5. **Container isolation** — Each service runs in its own Docker container with defined network boundaries.
6. **VPS as exit node** — Foreign VPS provides jurisdictional separation for internet traffic when needed.

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Intel i5 over N100 | 4x 4K + 2x 1080p decode with all services needs headroom. N100 is too tight. |
| Coral USB over NPU | Proven Frigate compatibility. NPU support is immature and risks instability. |
| Notify-only Watchtower | Client deployment must be stable. Updates tested on operator's stack first. |
| Dual Pi-hole instances | DNS filtering on both exit paths. No unfiltered DNS regardless of tunnel. |
| Unbound over upstream DNS | Full DNS privacy. No queries sent to third-party resolvers. |
| Managed PoE switch | VLAN support for camera network isolation. |
| Single Home Node | Simplifies maintenance, reduces hardware cost, single point of management. |
