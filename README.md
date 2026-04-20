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
| DB Snapshots | 80GB volume snapshot (+10GB free) | ~$5.60 | ~$67 |
| Object Storage | Tigris 250GB, zero egress | $5.00 | $60 |
| Load Balancer | Fly.io Anycast (built-in) | $0 | $0 |
| Caching | ETS/DETS in-process | $0 | $0 |
| **TOTAL** | | **$48.14–133.30** | **$578–1,600** |

**Annual cost: ~$578–1,600/year** (low traffic: zones idle; high traffic: both zones always on)

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

## Home User Deployment

Home users can run the full stack without a cloud provider using one of two
networking strategies, or both at once.

### Strategy A — Port forwarding HTTP/3 on port 443

WebTransport requires QUIC (UDP). Forward **UDP 443** on your router to the
machine running the zone server.

```
Internet
  │  UDP 443 (QUIC)
  ▼
Router  ─── UDP 443 forwarded ──▶  Zone server machine (picoquic)
```

#### DNS naming — zone + MAC digits schema

Each zone server is named `zone-<last4hex>.<your-domain>`, where `<last4hex>`
is the last 4 hex digits (2 bytes) of the machine's primary network interface
MAC address. This gives each machine a stable, collision-resistant subdomain
derived from hardware identity — no manual name assignment needed.

```bash
# Get last 4 hex digits of the primary interface MAC (Linux)
MAC=$(ip link show $(ip route show default | awk '/dev/{print $5}' | head -1) \
      | awk '/link\/ether/{print $2}')
LAST4=$(echo "$MAC" | tr -d ':' | tail -c 5)
echo "zone-${LAST4}.example.com"
# e.g. MAC aa:bb:cc:dd:70:0a → zone-700a.example.com
```

Add an A record in your DNS provider:

```
zone-<last4hex>.<your-domain>  A  <your-public-IP>
```

Set in `.env` (see `multiplayer-fabric-hosting/.env.example`):

```bash
ZONE_HOST=zone-<last4hex>.<your-domain>
ZONE_PORT=443
```

#### Steps

1. **Router rule:** forward UDP port 443 → zone server LAN IP.
2. **DNS record:** create `zone-<last4hex>.<your-domain>` A record pointing at
   your public IP. Use Cloudflare free DNS or duckdns with a DDNS update script
   if your IP changes.
3. **Set ZONE_HOST:** put the full hostname in `.env` so the zone server
   advertises the correct address in its self-signed cert and Uro registration.
4. **TLS:** the zone server generates a self-signed cert on startup and stores
   its fingerprint in Uro (`cert_hash`). Clients pin to that hash — no CA
   needed.
5. **Register the shard** in zone_console:
   ```
   register zone-<last4hex>.<your-domain> 443 <map> <name>
   ```

Clients connect directly over QUIC — no proxy, lowest possible latency.
Requires your ISP to not block inbound UDP 443 (most residential ISPs allow
it; some CGNAT setups do not).

### Strategy B — Cloudflare Tunnel HTTP/1 (no port forwarding)

Cloudflare Tunnel (`cloudflared`) creates an outbound-only TCP connection to
Cloudflare's edge — no inbound firewall rules or port forwarding required.
The tunnel only proxies HTTP/1.1 and HTTP/2 (TCP); it cannot carry QUIC/UDP,
so **WebTransport zone traffic cannot use this path**.

Use the tunnel for the **Uro backend** (REST API, HTTP/1.1) and serve zone
connections separately if UDP reachability is available.

```
Internet
  │  HTTPS (HTTP/1.1)
  ▼
Cloudflare edge ─── tunnel ──▶  cloudflared ──▶  Uro backend (Phoenix)

Zone server: not reachable via tunnel — requires Strategy A or STUN/TURN.
```

Steps:

1. Install cloudflared and authenticate:
   ```bash
   cloudflared tunnel login
   cloudflared tunnel create multiplayer-fabric
   ```
2. Create the tunnel config (`~/.cloudflared/config.yml`):
   ```yaml
   tunnel: <tunnel-id>
   credentials-file: /home/<user>/.cloudflared/<tunnel-id>.json
   ingress:
     - hostname: uro.example.com
       service: http://localhost:4000
     - service: http_status:404
   ```
3. Add a CNAME in Cloudflare DNS: `uro.example.com → <tunnel-id>.cfargotunnel.com`.
4. Run: `cloudflared tunnel run multiplayer-fabric`
5. Set `URO_BASE_URL=https://uro.example.com` in zone_console.

Zone servers remain unreachable for WebTransport unless Strategy A is also
applied. If your ISP uses CGNAT (no public IP), Strategy B covers Uro and
zone WebTransport is unavailable without a relay (TURN/VPN).

### Mixing both strategies

| Component | Strategy A | Strategy B |
| --------- | ---------- | ---------- |
| Uro backend (REST) | DDNS + TCP 443 forwarded | Cloudflare Tunnel ✓ |
| Zone server (WebTransport) | UDP 443 forwarded ✓ | Not supported |

The recommended home setup is **Strategy B for Uro + Strategy A for zones**:
Cloudflare Tunnel handles the REST API with zero port-forwarding and automatic
HTTPS; UDP 443 forwarding handles WebTransport zone traffic directly.

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
