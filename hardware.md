# PLEROMA — Hardware Specification & Bill of Materials

> ⚠️ **Pricing as of February 2026 (US market).** Hardware procurement targeted for ~May 2026. Prices and model availability will change. This document defines the **spec requirements** — specific models are recommendations, not mandates. Buy the best value that meets the spec when ready.

## Home Node — Mini PC

### Required Specifications

| Spec | Minimum | Recommended | Rationale |
|------|---------|-------------|-----------|
| CPU | Intel i5-12th gen | Intel i5-12450H / i5-1240P / i5-1340P | Quick Sync for hardware video decode. 8+ threads for concurrent services. |
| RAM | 16GB | 32GB DDR4/DDR5 | Frigate + 10 containerized services. 16GB will hit limits under load. |
| Storage (boot) | 256GB NVMe | 512GB NVMe | OS, Docker images, volumes, Prometheus TSDB, Grafana databases. |
| Storage (recordings) | 4TB | 4TB NVMe or 8TB HDD | See storage calculations below. |
| M.2 slots | 1 | 2 (dual NVMe) | Boot + recordings on separate drives. Cleaner, better I/O separation. |
| USB 3.0+ | 1 port minimum | 2+ ports | Coral USB Accelerator + headroom. |
| Ethernet | 1x Gigabit | 1x 2.5GbE | 6 camera streams + all services. 2.5GbE provides headroom. |
| WiFi | Not required | Optional | All critical connections are wired. |

### Recommended Models (US Market, Feb 2026 Reference Pricing)

| Model | CPU | RAM | Storage | Est. Price | Notes |
|-------|-----|-----|---------|------------|-------|
| Beelink SEi12 Pro | i5-12450H | 16GB (upgrade to 32) | 500GB NVMe | $280–350 | Dual NVMe slots. Solid build. |
| Minisforum NAB6 | i5-12450H | 16GB (upgrade to 32) | 512GB NVMe | $300–380 | Dual NVMe. Good thermal design. |
| GMKtec NucBox K6 | i5-12450H | 16GB (upgrade to 32) | 512GB NVMe | $260–330 | Budget-friendly. Verify dual NVMe. |
| Minisforum UM690S | Ryzen 9 6900HX | 32GB | 512GB NVMe | $350–450 | AMD alternative. No Quick Sync — uses VAAPI. |

> **Selection criteria when purchasing**: Prioritize dual NVMe slots, verified 32GB RAM support, and USB 3.0 availability. Check current reviews for thermal throttling issues. Intel preferred for Frigate Quick Sync maturity.

### RAM Upgrade

Most mini PCs in this class ship with 16GB. Budget for a 32GB SO-DIMM kit (~$50-80) if the unit doesn't come with 32GB.

---

## AI Accelerator

| Item | Model | Est. Price | Notes |
|------|-------|------------|-------|
| AI detection | Google Coral USB Accelerator | $30–40 | The standard for Frigate. Proven, reliable, low power. |

> **Why not an integrated NPU?** Intel Core Ultra and AMD Ryzen AI chips have built-in NPUs, but Frigate support is still experimental. For a client deployment where stability is the priority, the Coral USB is the correct choice. Revisit NPU options in future iterations.

---

## Cameras

### Outdoor — 4K PoE (x4)

| Model | Resolution | Est. Price (each) | Notes |
|-------|------------|-------------------|-------|
| Reolink RLC-810A | 4K / 8MP | $55–65 | Best Frigate compatibility. Reliable RTSP. Excellent community support. |
| Amcrest IP8M-T2669EW | 4K / 8MP | $60–75 | Strong alternative. Good night vision. |

### Indoor — PoE (x2)

| Model | Resolution | Est. Price (each) | Notes |
|-------|------------|-------------------|-------|
| Reolink RLC-520A | 1080p / 5MP | $35–45 | Same ecosystem as outdoor. Consistent configuration. |
| Reolink RLC-510A | 4MP | $30–40 | Budget option. Still good Frigate support. |

> **Recommendation**: Stick with Reolink for all cameras. Same RTSP config pattern, same firmware ecosystem, simplifies maintenance. The RLC-810A and RLC-520A are the most documented cameras in the Frigate community.

### Camera Stream Configuration (Frigate)

Each camera provides two RTSP streams:

| Stream | Use | Resolution | Bitrate (est.) |
|--------|-----|------------|-----------------|
| Main stream | Recording | 4K (outdoor) / 1080p (indoor) | 8–12 Mbps (4K) / 3–5 Mbps (1080p) |
| Sub stream | AI detection | 720p | 1–2 Mbps |

Frigate uses the sub stream for Coral-based object detection (low overhead) and records the main stream (full quality). This is critical — the Coral is not processing 4K frames.

---

## Storage Calculations

### Recording Bandwidth

| Source | Count | Resolution | Bitrate (each) | Total |
|--------|-------|------------|-----------------|-------|
| Outdoor cameras | 4 | 4K H.265 | ~10 Mbps | 40 Mbps |
| Indoor cameras | 2 | 1080p H.265 | ~4 Mbps | 8 Mbps |
| **Total** | **6** | | | **48 Mbps** |

### Daily Storage

```
48 Mbps × 3600 sec/hr × 24 hr = 4,147,200 Mb/day
4,147,200 / 8 = 518,400 MB/day
≈ 507 GB/day (continuous recording, all cameras)
```

### Retention Matrix

| Retention | Storage Required | Recommended Drive |
|-----------|-----------------|-------------------|
| 7 days | ~3.5 TB | 4TB NVMe |
| 10 days | ~5.1 TB | 8TB HDD or 2x 4TB |
| 14 days | ~7.1 TB | 8TB HDD |
| 30 days | ~15.2 TB | Not practical for single-drive |

### Recommendation

**Target: 7–10 day retention.**

- **Option A (clean)**: 4TB NVMe in second M.2 slot → ~7–8 days retention. No external drives. Silent. Fast. ~$200-300.
- **Option B (maximum)**: 8TB HDD via USB 3.0 enclosure → 10–14 days retention. Cheap. Slightly noisier. ~$120-150 for drive + $15-20 for enclosure.
- **Option C (hybrid)**: 4TB NVMe for recent recordings + 8TB HDD for overflow/archive. Maximum flexibility. ~$350-450 total.

Frigate handles retention automatically — it will delete oldest recordings when disk reaches configured threshold.

---

## Network Infrastructure

### PoE Switch

| Model | Ports | PoE Ports | PoE Budget | Managed | Est. Price | Notes |
|-------|-------|-----------|------------|---------|------------|-------|
| TP-Link TL-SG108PE | 8 | 4 PoE+ | 64W | Smart | $60–70 | Minimum viable. 4 PoE ports limits expansion. |
| TP-Link TL-SG2210P | 8+2 SFP | 8 PoE+ | 150W+ | L2 Managed | $100–130 | VLAN support. Camera isolation. Professional. |
| Netgear GS310TP | 8 | 8 PoE+ | 55W | Smart | $90–110 | Alternative managed option. |

> **Recommendation**: TP-Link TL-SG2210P or equivalent managed switch with VLAN support. Camera VLAN isolation is a meaningful security feature and a good selling point for the project. Budget the extra $30-50.

### Cabling

| Item | Quantity | Est. Price | Notes |
|------|----------|------------|-------|
| Cat6 bulk cable (outdoor rated) | 500 ft / 150m | $60–90 | CMX or direct burial rated for outdoor runs. |
| Cat6 bulk cable (indoor) | 250 ft / 75m | $25–40 | Standard CMR/CMP rated. |
| RJ45 connectors (pass-through) | 50-pack | $10–15 | Pass-through style recommended for easier crimping. |
| RJ45 crimp tool | 1 | $20–30 | If not already owned. |
| Cable tester | 1 | $15–25 | Essential for verifying custom crimps. |
| Cable clips / conduit | Assorted | $15–25 | Outdoor cable management. |

> **Note**: Daniel will be doing physical camera installation and custom cable crimping.

---

## Foreign VPS

| Provider | Plan | vCPU | RAM | Storage | Est. Price | Notes |
|----------|------|------|-----|---------|------------|-------|
| HostHatch | Entry VPS | 1+ vCPU | 512MB–1GB | 10–20GB SSD | $35–50/yr | Running WireGuard + Pi-hole + Unbound only. Minimal resources needed. |

> VPS selection based on Daniel's existing HostHatch infrastructure experience. Same provider simplifies management.

---

## Full Bill of Materials — Summary

| Category | Item | Est. Cost (USD) |
|----------|------|----------------|
| **Compute** | Mini PC (i5-12450H, 32GB, 512GB NVMe) | $300–400 |
| | RAM upgrade to 32GB (if needed) | $50–80 |
| | Google Coral USB Accelerator | $30–40 |
| **Storage** | 4TB NVMe (recordings) OR 8TB HDD | $200–300 / $120–150 |
| **Cameras** | 4x Reolink RLC-810A (outdoor 4K) | $220–260 |
| | 2x Reolink RLC-520A (indoor) | $70–90 |
| **Network** | Managed PoE+ switch (8-port) | $100–130 |
| | Cat6 cable (bulk, outdoor + indoor) | $85–130 |
| | RJ45 connectors, crimp tool, tester | $45–70 |
| | Cable management / conduit | $15–25 |
| **VPS** | HostHatch annual | $35–50/yr |
| | | |
| **Estimated Total** | | **$1,050–1,575** |
| **Estimated Annual Recurring** | VPS only | **$35–50/yr** |

> These are planning estimates. Final BOM to be priced at time of procurement (~May 2026).

---

## Pre-Purchase Checklist

Before buying, verify:

- [ ] Mini PC has dual M.2 NVMe slots (check teardown/review videos)
- [ ] Mini PC RAM is upgradeable to 32GB (check supported SO-DIMM specs)
- [ ] Mini PC has at least 1x USB 3.0 port for Coral (2x preferred)
- [ ] Mini PC has at least 1x Gigabit Ethernet (2.5GbE preferred)
- [ ] Coral USB Accelerator is in stock (availability fluctuates)
- [ ] Camera models still current (Reolink refreshes product lines periodically)
- [ ] PoE switch PoE budget sufficient for 6 cameras (~60-80W total)
- [ ] VPS provider plan still available at expected price
- [ ] Cat6 cable rating appropriate for outdoor runs (CMX/direct burial)
