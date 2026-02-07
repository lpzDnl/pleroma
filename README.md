# P.L.E.R.O.M.A.

**Privacy-Layered Environment for Resilient Operations, Monitoring & Access**

> *"The Pleroma is everything and nothing. It is the fullness."*
> — Carl Jung, *The Red Book*

---

## Overview

PLEROMA is a self-hosted privacy and security infrastructure stack, designed as a single deployable system for home networks. It combines AI-powered video surveillance, encrypted remote access, network-wide ad/tracker blocking, private search, and full-stack monitoring — all containerized, all under your control.

No cloud dependencies. No subscriptions. No data leaves your network unless you say so.

## Architecture

PLEROMA spans two deployment targets:

| Target | Role | Services |
|--------|------|----------|
| **Home Node** | Primary infrastructure (mini PC) | Frigate, WireGuard, Pi-hole, Unbound, SearXNG, Grafana, Prometheus, Portainer, Watchtower, Uptime Kuma, Homepage |
| **Foreign VPS** | Encrypted exit node + DNS filtering | WireGuard, Pi-hole, Unbound |

The two nodes are linked via a WireGuard tunnel, providing both a home VPN (direct access to home services from anywhere) and a foreign VPN (encrypted internet exit through a non-local jurisdiction).

See [docs/architecture.md](docs/architecture.md) for full network topology and service interconnections.

## Service Stack

| Service | Purpose | Deployment |
|---------|---------|------------|
| **Frigate** | AI-powered NVR with object detection (person, vehicle, animal) | Home |
| **WireGuard** | VPN tunnels — home access + foreign exit | Both |
| **Pi-hole** | DNS-level ad/tracker blocking | Both |
| **Unbound** | Recursive DNS resolver (no upstream DNS dependency) | Both |
| **SearXNG** | Privacy-respecting metasearch engine | Home |
| **Grafana** | Monitoring dashboards and visualization | Home |
| **Prometheus** | Metrics collection and time-series database | Home |
| **Portainer** | Docker container management UI | Home |
| **Watchtower** | Container update monitoring (notify-only mode) | Home |
| **Uptime Kuma** | Service health monitoring and alerting | Home |
| **Homepage** | Unified dashboard for all services | Home |

## Hardware Requirements

See [docs/hardware.md](docs/hardware.md) for detailed specifications, recommended models, storage calculations, and full bill of materials.

**Summary — Home Node:**
- Mini PC: Intel i5-12th/13th gen, 32GB RAM, dual NVMe slots
- Google Coral USB Accelerator (Frigate AI detection)
- 4x 4K PoE outdoor cameras
- 2x 1080p PoE indoor cameras
- 8-port PoE+ managed switch
- 4TB+ NVMe or 8TB HDD for recording storage

**Summary — Foreign VPS:**
- Lightweight VPS (HostHatch or similar)
- 1+ vCPU, 512MB+ RAM sufficient
- Low storage requirements

## Project Status

> ⏳ **Planning Phase** — Hardware procurement targeted for ~May 2026

- [x] Architecture design
- [x] Service stack selection
- [x] Hardware specification and BOM
- [ ] Hardware procurement
- [ ] Network infrastructure (cabling, PoE switch, cameras)
- [ ] Base OS installation and hardening
- [ ] Docker Compose — Home Node
- [ ] Docker Compose — Foreign VPS
- [ ] Service configuration and tuning
- [ ] Frigate camera integration and zone configuration
- [ ] WireGuard tunnel establishment (Home ↔ VPS)
- [ ] Monitoring and alerting setup
- [ ] Security hardening and penetration testing
- [ ] Client device provisioning (WireGuard profiles, browser shortcuts)
- [ ] Documentation and runbooks
- [ ] Stability validation (30-day burn-in)

## Repository Structure

```
pleroma/
├── README.md
├── docs/
│   ├── architecture.md      # Network topology and service map
│   ├── hardware.md           # Specs, models, BOM, storage math
│   ├── services.md           # Service reference — ports, dependencies, config notes
│   └── maintenance.md        # Update strategy, backup, troubleshooting
├── home/
│   ├── README.md             # Home node deployment guide
│   ├── docker-compose.yml    # [Future] Home node compose file
│   └── configs/              # Service configuration files
│       ├── frigate/
│       ├── pihole/
│       ├── unbound/
│       ├── wireguard/
│       ├── searxng/
│       ├── grafana/
│       ├── prometheus/
│       ├── portainer/
│       ├── uptime-kuma/
│       ├── homepage/
│       └── watchtower/
├── vps/
│   ├── README.md             # VPS deployment guide
│   ├── docker-compose.yml    # [Future] VPS compose file
│   └── configs/              # Service configuration files
│       ├── pihole/
│       ├── unbound/
│       └── wireguard/
└── scripts/
    └── README.md             # Utility scripts (health checks, backups, etc.)
```

## License

TBD — License to be determined before public release.

## Author

Daniel — [GitHub Profile]

---

*PLEROMA: The fullness, self-contained.*
