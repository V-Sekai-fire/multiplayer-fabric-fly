# multiplayer-fabric-fly - Fly.io Deployment Design

## Architecture Overview

This document details the deployment architecture for V-Sekai's multiplayer fabric on Fly.io infrastructure, designed for optimal performance in West Coast Canada while minimizing costs through auto-stop machines and right-sized VMs.

## Component Sizing and Deployment Strategy

### 1. Zone Backend (Elixir/Phoenix — `multiplayer-fabric-zone-backend`)
- **Size:** 1x shared-cpu-1x, 512MB RAM
- **Region:** Toronto (yyz)
- **Auto-stop:** No (always-on for API availability)
- **Reasoning:** Handles REST API, user authentication, and zone management. Caddy runs on the same machine for TLS termination.
- **Monthly Cost:** $3.32 (~$40/year)
- **Justification:** Shared CPU is sufficient for Elixir/Phoenix at low traffic. ETS/DETS handle caching in-process with no extra service.

### 2. Zone Servers (Godot)
- **Size:** 2x performance-1x, 4GB RAM, 2 vCPU
- **Region:** Toronto (yyz)
- **Auto-stop:** Yes — machines scale to 0 when no players are connected
- **Min machines running:** 0
- **Reasoning:** Each zone runs a single headless Godot instance pinned to 1 core, with the second core reserved for the OS. Scale-to-zero means zones cost $0 when idle.
- **Monthly Cost:** $0–85.16 depending on uptime ($0–1,022/year) — $42.58/machine × 2
- **Justification:** performance-1x provides dedicated CPU for consistent game simulation. Scale-to-zero eliminates idle compute cost entirely; cold-start latency (~1–2s) is acceptable when a player join triggers the wake.

### 3. Database (CockroachDB)
- **Size:** 1x shared-cpu-2x, 4GB RAM
- **Region:** Toronto (yyz)
- **Volume:** 80GB persistent volume
- **Auto-stop:** No (always-on for database availability)
- **Reasoning:** Single-node CRDB retains the full SQL interface and tooling. Scale to 3-node when HA is required.
- **Monthly Cost:** $34.22 (~$411/year) — $22.22 machine + $12.00 volume (80GB × $0.15/GB)
- **Justification:** Single-node CRDB on a shared Fly machine costs far less than a managed database. Daily volume snapshots provide recovery path.

### 4. Object Storage
- **Service:** Tigris (Fly.io native S3-compatible storage)
- **Size:** 250GB
- **Reasoning:** Tigris is integrated with Fly.io, S3-compatible, and includes global CDN by default at no extra charge.
- **CDN Included:** Yes
- **Monthly Cost:** $5/month (~$60/year) — $0.02/GB, zero egress fees, 5GB free tier

### 5. Load Balancer
- **Solution:** Fly.io Anycast proxy (built-in)
- **Reasoning:** Fly.io routes traffic via its global Anycast network to the nearest machine automatically. No separate load balancer needed.
- **Monthly Cost:** $0

### 6. Caching / Session Storage
- **Solution:** ETS (Erlang Term Storage) or DETS (Disk-based ETS) in-process on the Uro backend
- **Reasoning:** ETS provides fast in-memory key/value storage built into the BEAM VM; DETS persists to disk and survives node restarts. Both require no separate service.
- **Monthly Cost:** $0
- **Justification:** ETS/DETS are faster than Redis for local reads with no network hop. Use ETS for ephemeral cache, DETS when durability across restarts is needed.

## Cost Analysis

| Component | Details | Monthly | Per Year |
|-----------|---------|---------|----------|
| Zone Backend | 1x shared-cpu-1x 512MB | $3.32 | $40 |
| Zone Servers | 2x performance-1x 4GB (auto-stop) | $0–85.16 | $0–1,022 |
| Database | 1x shared-cpu-2x 4GB + 80GB volume | $34.22 | $411 |
| Object Storage | Tigris 250GB, zero egress | $5.00 | $60 |
| Load Balancer | Fly.io Anycast (built-in) | $0 | $0 |
| Caching | ETS/DETS in-process | $0 | $0 |
| **TOTAL** | | **$42.54–127.70** | **$511–1,533** |

**Annual cost: ~$511–1,533/year** (low traffic: zones idle; high traffic: both zones always on)

Prices sourced from [Fly.io pricing](https://fly.io/docs/about/pricing/) and [Tigris pricing](https://www.tigrisdata.com/docs/pricing/) — April 2026.

**Scale-up triggers:** add a 3rd zone at ~60 concurrent players; scale CRDB to 3-node when HA is required; add a second Uro machine when API becomes the bottleneck.

## Architecture Decisions

1. **Fly.io Anycast over explicit Load Balancer:** Fly.io's proxy layer handles routing and TLS automatically. Caddy on the Uro machine handles local TLS termination for the Phoenix endpoint.

2. **Scale-to-zero Zone Machines:** `min_machines_running = 0` means Fly.io fully stops all zone machines when no players are connected — zero compute cost at idle. Cold-start latency (~1–2s) is acceptable since a player join triggers the wake.

3. **Single-node CRDB on Fly Volume:** Retains CockroachDB's SQL dialect and future 3-node upgrade path. Persistent volumes survive machine restarts. Daily snapshots via `fly volumes snapshot` provide backup.

4. **Tigris for Object Storage:** Native Fly.io integration with S3-compatible API and built-in CDN — replaces DO Spaces without changing application code.

5. **ETS/DETS instead of Redis:** Session and cache data live in-process on the BEAM. No network hop, no extra service to operate.

## Deployment Instructions

1. **Deploy Uro Backend:**
   ```bash
   fly launch --vm-size shared-cpu-1x --vm-memory 512 --region yyz
   ```

2. **Configure Zone Servers:**
   ```bash
   fly machine create --app multiplayer-fabric-zones --region yyz \
     --vm-size performance-1x \
     --vm-memory 4096 \
     --auto-stop-timeout 30 \
     --image "ghcr.io/v-sekai-fire/godot-server:latest"
   ```

3. **Database Setup:**
   ```bash
   # Create persistent volume
   fly volumes create crdb_data --region yyz --size 80

   # Launch single-node CockroachDB
   fly machine create --app multiplayer-fabric-crdb --region yyz \
     --vm-size shared-cpu-2x \
     --vm-memory 4096 \
     --volume crdb_data:/cockroach/cockroach-data \
     --image cockroachdb/cockroach:latest \
     --port 26257
   ```

4. **Environment Configuration:**
   ```bash
   fly secrets set DATABASE_URL="postgresql://user:pass@<crdb-host>:26257/production"

   # Tigris bucket (created via fly storage create)
   fly secrets set AWS_S3_BUCKET="game-data" \
     AWS_S3_ENDPOINT="https://fly.storage.tigris.dev" \
     AWS_ACCESS_KEY_ID="[...]" \
     AWS_SECRET_ACCESS_KEY="[...]"
   ```

5. **Monitoring Setup:**
   ```bash
   fly logs --app multiplayer-fabric-crdb
   fly status --app multiplayer-fabric-zones
   ```

---
Test with a series of small deployments to finalize configuration before production.

> **Note:** Focus on safe defaults first — auto-scaling over replication strategies except for very stable, predictable workloads.

## References

```bibtex
@article{zhang2025verbalized,
  title   = {Verbalized Sampling: How to Mitigate Mode Collapse and Unlock {LLM} Diversity},
  author  = {Zhang, Jiayi and Yu, Simon and Chong, Derek and Sicilia, Anthony and Tomz, Michael R. and Manning, Christopher D. and Shi, Weiyan},
  journal = {arXiv preprint arXiv:2510.01171},
  year    = {2025},
  url     = {https://arxiv.org/abs/2510.01171}
}
```
