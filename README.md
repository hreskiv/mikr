# MikroTik Manager

Self-hosted web application for managing MikroTik device fleets. Monitor, configure, upgrade, and backup your devices from a single dashboard with real-time WebSocket updates.

[![Version](https://img.shields.io/badge/version-1.5.8-blue)](https://github.com/hreskiv/mikr/releases)
[![Docker](https://img.shields.io/badge/docker-ghcr.io%2Fhreskiv%2Fmikr-blue)](https://ghcr.io/hreskiv/mikr)

## Screenshots

<!-- TODO: add screenshots -->
*Screenshots coming soon — dashboard, device detail with port visualization, command execution, config diff comparison.*

## Features

### Monitoring
- **Real-time dashboard** — device status cards with WebSocket live updates (60s polling)
- **SNMP monitoring** — lightweight SNMPv2c polling (CPU, memory, uptime, temperature, voltage)
- **Three connection methods** — SSH, REST API, or SNMP-only per device
- **SNMP as supplementary** — SSH/REST devices can also use SNMP for faster status checks

### Device Management
- **Site grouping** — organize devices by physical location
- **Device tags** — assign tags with autocomplete, filter by multiple tags (Shift+click)
- **Bulk editing** — select multiple devices, change connection parameters in one action
- **Enable/disable** — disabled devices skip monitoring, dimmed in UI
- **Import from scan** — discover and add devices from network scan results

### Operations
- **Bulk CLI commands** — execute on multiple devices with live WebSocket output
- **RouterOS upgrades** — check for updates + upgrade with real-time progress
- **Firmware upgrades** — write firmware + automatic reboot
- **Config backup & export** — save device configurations to database
- **Side-by-side diff** — compare any two backups visually
- **Backup scheduling** — automated exports with time-of-day selection and flexible intervals (2h to 7d)

### Network Discovery
- **IP range scanning** — CIDR, dash ranges, single IP (probes SSH + HTTPS + HTTP)
- **Neighbor discovery** — MNDP / LLDP / CDP with clickable links to managed devices
- **MAC→IPv4 cross-reference** — resolves link-local IPv6 neighbors to real addresses

### Visualization
- **Full interface discovery** — all types: ethernet, SFP, bridge, VLAN, bonding, wireless, WireGuard, EoIP, GRE, PPPoE
- **Grouped interface view** — Ethernet/Physical, CAPsMAN/WiFi, Bridge/VLAN/Bond, Tunnels/VPN, categorized with chips
- **Physical port grid** — colored squares by link speed (10M / 100M / 1G / 10G)
- **PoE indicators** — lightning bolt icon with power, voltage, current in tooltip
- **DHCP leases** — view all leases with IP, MAC, hostname, status badges, and expiry time
- **Wireless clients** — connected clients with signal strength, TX/RX rates, uptime, and IP from DHCP
- **Auto-refresh** — DHCP and wireless tables update every 30s while visible, disconnected clients disappear automatically
- **IP services** — see all MikroTik services (SSH, API, WWW, Winbox, FTP) as colored pills, toggle enable/disable with safety checks
- **Route counting** — per-protocol breakdown (static, connected, BGP, OSPF, RIP, etc.)

### Security & Access
- **Role-based access** — admin / operator / viewer
- **JWT authentication** — access token (15min) + refresh token (7d)
- **Encrypted passwords** — AES-256-GCM for stored device credentials
- **Dark / Light theme** — toggle in sidebar, persisted in localStorage

## Quick Start

```bash
# Pull image
docker pull ghcr.io/hreskiv/mikr:latest

# Create project directory
mkdir -p /opt/mikr/data && cd /opt/mikr

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  mikr:
    image: ghcr.io/hreskiv/mikr:latest
    container_name: mikr-manager
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./data:/app/data
    environment:
      - PORT=3000
      - HOST=0.0.0.0
      - STORAGE_ADAPTER=sqlite
      - JWT_SECRET=change-me-to-a-long-random-string
      - ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
EOF

# Start
docker compose up -d

# Create default admin user (first run only)
docker exec mikr-manager node scripts/seed.js
```

Open `http://<host>:3000`, login: **admin** / **admin**

> **Production:** Change the default password immediately. Create a dedicated MikroTik user group with only the required policies instead of using `admin` with full access:
> ```
> /user/group/add name=manager-group policy=ssh,reboot,read,write,sensitive,rest-api,!local,!telnet,!ftp,!policy,!test,!winbox,!password,!web,!sniff,api,!romon
> /user/add name=mikr group=manager-group password=YOUR_PASSWORD
> ```
> This limits the blast radius if the manager is compromised.

## Without Docker Compose

```bash
docker pull ghcr.io/hreskiv/mikr:latest
mkdir -p /opt/mikr/data

docker run -d \
  --name mikr-manager \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /opt/mikr/data:/app/data \
  -e PORT=3000 \
  -e HOST=0.0.0.0 \
  -e STORAGE_ADAPTER=sqlite \
  -e JWT_SECRET=$(openssl rand -hex 32) \
  -e ENCRYPTION_KEY=$(openssl rand -hex 32) \
  ghcr.io/hreskiv/mikr:latest

docker exec mikr-manager node scripts/seed.js
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JWT_SECRET` | Yes | `dev-secret-change-me` | JWT signing key (min 32 chars) |
| `ENCRYPTION_KEY` | Yes | built-in default | 64 hex chars (32 bytes). Generate: `openssl rand -hex 32` |
| `PORT` | No | `3000` | Server port |
| `HOST` | No | `0.0.0.0` | Bind address |
| `STORAGE_ADAPTER` | No | `sqlite` | Storage backend |
| `LOG_LEVEL` | No | `info` | Log level |
| `MONITOR_INTERVAL_MS` | No | `60000` | Device polling interval (ms) |
| `MONITOR_CONCURRENCY` | No | `10` | Parallel device checks |

**Important:** `ENCRYPTION_KEY` must be exactly 64 hex characters. Device passwords are encrypted with this key — if changed, existing passwords won't decrypt.

## Connection Methods

| Capability | SSH | REST API | SNMP |
|------------|-----|----------|------|
| Status monitoring | ✓ | ✓ | ✓ |
| CLI commands | ✓ | ✓ | — |
| RouterOS upgrades | ✓ | ✓ | — |
| Config backup | ✓ | ✓ | — |
| Interface details | ✓ | ✓ | ✓ |
| Self-signed TLS | N/A | ✓ | N/A |
| Session overhead | 1 SSH conn | HTTPS per request | UDP per poll |

- **SSH** — non-interactive exec, best compatibility with all RouterOS versions
- **REST API** — available on RouterOS 7.1+, supports HTTPS and HTTP
- **SNMP** — monitoring only (no commands, upgrades, or backups). Useful for devices where SSH/REST isn't available

Devices can use SSH or REST as primary method, with SNMP as an optional supplementary source for faster status checks. The device detail page detects available methods from actual service data and allows quick switching between them.

## Licensing

All features are available on every tier — the only difference is the device limit.

| | Community | License 50 | License Unlimited |
|---|-----------|------------|------------------|
| **Devices** | up to 10 | up to 50 | Unlimited |
| **Price** | €0 | €149 (one-time) | €399 (one-time) |
| **Updates** | — | €49/year (optional) | €99/year (optional) |
| **All features** | ✓ | ✓ | ✓ |

- **Perpetual license** — the software works forever on the purchased version
- **Update subscription** — grants access to new versions (optional, not required)
- **Self-hosted** — your data stays on your server, offline license activation
- No account required, no phone-home, no telemetry

> During the beta period, all installations run with unlimited access.

## Data & Backup

SQLite database stored in `./data/mikr.db`. Persists across container updates.

```bash
# Backup
cp data/mikr.db mikr-backup.db

# Restore
cp mikr-backup.db data/mikr.db
docker restart mikr-manager
```

## Management

```bash
docker compose logs -f       # View logs
docker compose restart       # Restart
docker compose down          # Stop
docker compose up -d         # Start / update
```

## Tech Stack

- **Backend**: Node.js 22, Express.js, SQLite (better-sqlite3)
- **Frontend**: Vanilla JS SPA (no framework), CSS3
- **Real-time**: WebSocket for live status, command output, upgrade progress
- **Auth**: JWT + bcrypt, role-based (admin / operator / viewer)
- **MikroTik**: SSH (node-ssh) + REST API (https module) + SNMP (net-snmp)
- **Encryption**: AES-256-GCM for stored device passwords
- **Container**: Docker, node:22-alpine (multi-stage build)
