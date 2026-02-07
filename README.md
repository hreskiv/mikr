# MikroTik Manager

Web-based MikroTik device management system. Monitor, configure, and upgrade your MikroTik fleet from a single dashboard.

## Features

- **Dashboard** — real-time device status monitoring (polling every 60s)
- **Device management** — add, edit, delete, group by sites
- **Bulk commands** — execute CLI commands on multiple devices simultaneously
- **RouterOS & firmware upgrades** — with live WebSocket progress
- **Config backup** — export with side-by-side diff comparison
- **Network scanning** — discover MikroTik devices in IP ranges
- **Interface ports** — physical port visualization with link speed colors, PoE status
- **Neighbor discovery** — MNDP/LLDP/CDP neighbors with links to managed devices
- **Route counting** — per-protocol breakdown (BGP, OSPF, static, connected)
- **User management** — role-based access (admin / operator / viewer)
- **SSH & REST API** — dual connection method support per device
- **Dark / Light theme**

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
> /user/group/add name=mikr-manager policy=read,write,ssh,rest-api,sensitive,reboot
> /user/add name=mikr group=mikr-manager password=YOUR_PASSWORD
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
- **MikroTik**: SSH (node-ssh) + REST API (https module)
- **Encryption**: AES-256-GCM for stored device passwords
- **Container**: Docker, node:22-alpine (multi-stage build)
