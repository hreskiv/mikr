# MikroTik Manager

Self-hosted web application for managing MikroTik device fleets. Monitor, configure, upgrade, and backup your devices from a single dashboard with real-time WebSocket updates.

[![Version](https://img.shields.io/badge/version-1.37.0-blue)](https://github.com/hreskiv/mikr/releases)
[![Docker](https://img.shields.io/badge/docker-ghcr.io%2Fhreskiv%2Fmikr-blue)](https://ghcr.io/hreskiv/mikr)

## Screenshots

<p align="center">
  <img src="screenshots/screenshot.png" alt="Dashboard — device status grouped by site" width="100%">
</p>

<details>
<summary>More screenshots</summary>

<p align="center">
  <img src="screenshots/screenshot-device-detail.png" alt="Device detail — interfaces, PoE, neighbors" width="100%">
</p>

<p align="center">
  <img src="screenshots/screenshot-upgrades.png" alt="Upgrades — sequential queue, version badges" width="100%">
</p>

<p align="center">
  <img src="screenshots/screenshot-backups.png" alt="Backups — side-by-side config diff" width="100%">
</p>

</details>

## Features

### Monitoring
- **Real-time dashboard** — device status cards with WebSocket live updates (60s polling, configurable per device)
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
- **Run-as credential override** — optionally run a bulk command under your own RouterOS login for that one execution; credentials are never stored or logged
- **Tag-based target selection** — toggle all devices carrying a tag on/off with one click, alongside Select All/Online and manual picking
- **RouterOS upgrades** — check for updates + upgrade with real-time progress
- **Firmware upgrades** — write firmware + automatic reboot
- **Config backup & export** — save device configurations to database
- **Side-by-side diff** — compare any two backups visually
- **Backup scheduling** — automated exports with time-of-day selection and flexible intervals (2h to 7d)

### Logging & Observability
- **Built-in syslog receiver (UDP + TCP)** — the Manager ships both UDP and TCP listeners on port `5514` that ingest MikroTik syslog messages, persist them to SQLite, and stream live to the UI over WebSocket. TCP is useful when UDP is blocked by firewalls or when you want guaranteed delivery — RouterOS 7.x supports `Remote Log Protocol: TCP` natively.
- **Native RouterOS format** — topics and messages appear exactly as in `/log print` (works out of the box with `remote-log-format=default`; also accepts `<PRI>`-prefixed and `bsd-syslog=yes` RFC3164 forms)
- **Unified Logs page** — all devices in one stream with colored severity stripe, filters (device, severity, topic, text search), time range selector (`All time` / `Last 1h` / `6h` / `24h` / `7d`), **Load older** pagination, pause and auto-scroll
- **Per-device Logs tab** — focused view on the device detail page for troubleshooting a specific router
- **Multi-IP device correlation** — logs are matched to the right device by **any** interface IP (collected each monitor poll), so the syslog source IP doesn't need to equal the management IP and `src-address=` pinning is optional
- **Per-device retention override** — raise the row cap on chatty core/border routers, lower it on quiet APs, so a log-storm on one device can't evict logs from the rest of the fleet
- **Setup Guide modal** — one click generates a copy-paste MikroTik CLI snippet and an optional **Command Template** to apply the config to every device at once
- **On-device syslog + RouterOS `dstnat` (if running mikr inside a MikroTik Container App)** — see the full install guide at [mikr.app/install.html#container](https://mikr.app/install.html#container)

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
- **IPsec tunnels** — configured peers with established/not established state, traffic counters, uptime
- **WireGuard peers** — endpoint, last handshake (color-coded by recency), TX/RX counters
- **IP services** — see all MikroTik services (SSH, API, WWW, Winbox, FTP) as colored pills, toggle enable/disable with safety checks
- **Route counting** — per-protocol breakdown (static, connected, BGP, OSPF, RIP, etc.)
- **Mobile-friendly layout (v1.36.0+)** — manage the fleet from a phone or tablet in the field: slide-in drawer menu, tables that reflow into cards or scroll horizontally, full-width bottom-sheet modals, a readable stacked log viewer, and touch-sized controls. Desktop layout unchanged; Topology stays desktop-only

### Security & Access
- **HTTPS / TLS** — optional HTTPS server on port 3443; auto-generated self-signed cert or bring your own; HTTP and HTTPS run in parallel; WebSocket (WSS) works automatically over HTTPS
- **Role-based access** — superadmin / admin / operator / viewer
- **Per-site access control (v1.33.0+)** — scope any non-superadmin user to specific sites; the role decides what they can do, the assigned sites decide where. Scoped users only see and act on their sites everywhere (dashboard, devices, logs, backups, live status, bulk commands/upgrades), enforced server-side. Superadmin manages users and grants site access; unrestricted users keep full-fleet access
- **JWT authentication** — access token (15min) + refresh token (7d)
- **RouterOS CVE alerting** — each device's RouterOS version is matched daily against the public NVD vulnerability feed. Dedicated **Security** page with severity/site filters, a per-device Security Advisories card, a `N CVE` badge on device cards, and a High/Critical dashboard banner. Each entry shows the CVSS score/vector, summary, advisory link, and the version that fixes it. Scoped to RouterOS 7; optional `cve-alert` webhook. Informational only — never blocks upgrades or commands
- **Auto-block / Simple IDS** — detects repeated failed logins from the syslog stream, aggregates them per source IP across the whole fleet, and past a threshold (default 5 / 10 min) blocks the IP by maintaining a `mikr-blocklist` firewall address-list with a native 72h timeout. Per-device opt-in; **Audit mode** (list candidates, block by hand) or **Auto-block** (block at threshold); whitelist (manual CIDRs + auto private-range + hard-whitelisted server IP). Address-list only — you add one drop rule that references the list; mikr never touches your firewall chains. Candidates / Blocked / Whitelist tabs on the Security page
- **GeoIP rule generator** — pick countries and generate an idempotent RouterOS firewall script sourced from ipdeny.com: **Blocklist** (drop the selected countries; inbound/outbound, `raw` or `filter`) or **Allowlist** (keep only the selected countries, drop the rest — auto-adds an established/private/always-allow safety block so it can't lock you out); IPv4/IPv6, optional weekly auto-refresh; saved as a Script command template and installed via the existing Deploy flow
- **Two-factor authentication (TOTP)** — opt-in per user, RFC 6238 compatible with Google Authenticator / Authy / 1Password / Microsoft Authenticator; AES-256-GCM-encrypted secrets; 8 single-use backup codes; admin reset for lost devices
- **Passkey / WebAuthn / FIDO2 sign-in** — additive second factor alongside TOTP: Touch ID, Face ID, Windows Hello, Apple/Google passkey, YubiKey. Multiple named passkeys per account with last-used info, admin reset, counter-regression check against cloned authenticators. Requires HTTPS on a real domain (WebAuthn spec forbids IP RP IDs) — LAN-IP installs see an explicit "Unavailable" notice rather than silent failure
- **Encrypted passwords** — AES-256-GCM for stored device credentials
- **Dark / Light theme** — toggle in sidebar, persisted in localStorage

### Configuration
- **Settings UI for runtime knobs (v1.30.0+)** — admin **Settings → System configuration**: 21 knobs across 11 groups (CVE feed enable/key/interval, activity-log retention, monitor poll cadence and concurrency, traffic and syslog retention, backup and upgrade scheduler intervals, JWT access/refresh expiry, WebAuthn RP ID / name / origins, log level, default SSH / REST API port for new devices). Edit live in the browser, no `docker-compose` edit or container restart, secrets encrypted at rest with AES-256-GCM. Sticky left subnav, one card per group, "from env" badge on any field locked by an environment variable.
- **Environment variables always win** — anything you set in `.env` / docker-compose stays the source of truth and renders as read-only in the UI with a "from env" badge. GitOps-friendly: existing compose-first deployments don't change behaviour after the upgrade. Bootstrap values (`PORT`, `HOST`, `JWT_SECRET`, `ENCRYPTION_KEY`, `TLS_*`, `SYSLOG_PORT`) stay env-only by design — they're either consumed before settings are loaded, or moving them would invalidate every stored secret on change.
- **Optional `.rsc` mirror to `/data/exports` (v1.30.0+, closes [#28](https://github.com/hreskiv/mikr/issues/28))** — toggle in Settings → External export. After each successful backup, also writes the RouterOS script export to `/data/exports/<site>/<device>.rsc` — one file per device, overwritten on each new backup. Mount `/data/exports` on a separate host volume for disaster recovery, point a git checkout at it for a free change-history (commit after each write), or rclone it to a NAS / cloud for 3-2-1 backups. Mikr never deletes from this volume; cleanup is yours (git rm / shell).

## Quick Start

> **Full install guide**, including how to deploy on a **MikroTik Container App** and configure the **RouterOS `dstnat` rule for UDP 5514** (syslog): see [mikr.app/install.html](https://mikr.app/install.html).

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
      - "3443:3443"    # HTTPS (optional, requires TLS_ENABLED=true)
      - "5514:5514/udp"  # Syslog UDP (optional — omit if not using the Logs page)
      - "5514:5514/tcp"  # Syslog TCP (optional — useful when UDP is blocked)
    volumes:
      - ./data:/app/data
    environment:
      - PORT=3000
      - HOST=0.0.0.0
      - STORAGE_ADAPTER=sqlite
      # JWT_SECRET and ENCRYPTION_KEY are auto-generated on first start and saved
      # to ./data/.secrets.json (mode 0600). Back up the data/ directory!
      # Override here if you want to set your own:
      # - JWT_SECRET=$(openssl rand -hex 48)
      # - ENCRYPTION_KEY=$(openssl rand -hex 32)
EOF

# Start
docker compose up -d

# Create default admin user (first run only)
docker exec mikr-manager node scripts/seed.js
```

Open `http://<host>:3000`, login: **admin** / **admin**

> **Production:** Change the default password immediately. Create a dedicated MikroTik user group with only the required policies instead of using `admin` with full access:
> ```
> /user/group/add name=manager-group policy=ssh,reboot,read,write,sensitive,rest-api,policy,!local,!telnet,!ftp,!test,!winbox,!password,!web,!sniff,api,!romon
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
  -p 5514:5514/udp \
  -p 5514:5514/tcp \
  -v /opt/mikr/data:/app/data \
  -e PORT=3000 \
  -e HOST=0.0.0.0 \
  -e STORAGE_ADAPTER=sqlite \
  ghcr.io/hreskiv/mikr:latest
# JWT_SECRET and ENCRYPTION_KEY are auto-generated and persisted to
# /opt/mikr/data/.secrets.json on first start. To use your own values,
# pass: -e JWT_SECRET=$(openssl rand -hex 48) -e ENCRYPTION_KEY=$(openssl rand -hex 32)

docker exec mikr-manager node scripts/seed.js
```

## Environment Variables

All variables below are optional with sensible defaults. **From v1.30.0, most of them are also editable from Admin → Settings → System configuration** without a compose edit or restart — when set as an env var here, they take precedence over the UI value and show as locked with a "from env" badge. Bootstrap values (marked **bootstrap** below) stay env-only.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JWT_SECRET` | No | auto-generated | Auto-generated on first start, persisted to `data/.secrets.json`. Override by setting env. Generate your own: `openssl rand -hex 48` |
| `ENCRYPTION_KEY` | No | auto-generated | 64 hex chars (32 bytes). Same auto-gen + persist as `JWT_SECRET`. Generate your own: `openssl rand -hex 32` |
| `PORT` | No | `3000` | Server port |
| `HOST` | No | `0.0.0.0` | Bind address |
| `STORAGE_ADAPTER` | No | `sqlite` | Storage backend |
| `LOG_LEVEL` | No | `info` | Log level |
| `MONITOR_INTERVAL_MS` | No | `60000` | Device polling interval (ms) |
| `MONITOR_CONCURRENCY` | No | `10` | Parallel device checks |
| `TLS_ENABLED` | No | `false` | Enable HTTPS server |
| `HTTPS_PORT` | No | `3443` | HTTPS server port |
| `TLS_CERT_PATH` | No | auto-generated | Path to custom TLS certificate (PEM) |
| `TLS_KEY_PATH` | No | auto-generated | Path to custom TLS private key (PEM) |
| `SYSLOG_ENABLED` | No | `true` | Enable the built-in UDP syslog receiver |
| `SYSLOG_PORT` | No | `5514` | UDP port for the syslog listener |
| `SYSLOG_TCP_ENABLED` | No | `true` | Enable the built-in TCP syslog receiver (parallel to UDP) |
| `SYSLOG_TCP_PORT` | No | `5514` | TCP port for the syslog listener (defaults to `SYSLOG_PORT`) |
| `SYSLOG_RETENTION_DAYS` | No | `7` | Drop log rows older than this many days |
| `SYSLOG_MAX_ROWS_PER_DEVICE` | No | `10000` | Global per-device row cap (overridable per device in the UI) |
| `CVE_ENABLED` | No | `true` | Enable RouterOS CVE alerting (daily NVD feed match) |
| `NVD_API_KEY` | No | _(none)_ | Optional NVD API key — raises the fetch rate limit (not required) |
| `CVE_CHECK_INTERVAL_MS` | No | `86400000` | How often to refresh the NVD feed (default 24h) |
| `IDS_ENABLED` | No | `false` | Enable Auto-block / Simple IDS (block source IPs of repeated failed logins; per-device opt-in, tune the rest in Settings) |
| `EXPORT_RSC_ENABLED` | No | `false` | Mirror the latest `.rsc` export to `/data/exports/<site>/<device>.rsc` after each backup (v1.30.0+). Mount `/data/exports` to a host volume for DR / git / rclone. |
| `ACTIVITY_LOG_RETENTION_DAYS` | No | `90` | Drop audit log rows older than this many days |
| `JWT_ACCESS_EXPIRY` | No | `15m` | Access token lifetime (jsonwebtoken duration: `15m`, `1h`, `30s`) |
| `JWT_REFRESH_EXPIRY` | No | `7d` | Refresh token lifetime |
| `WEBAUTHN_RP_ID` | No | _(none)_ | Registrable domain for passkey sign-in (e.g. `mikr.example.com`). **Must be a real domain, not an IP.** |
| `WEBAUTHN_RP_NAME` | No | `MikroTik Manager` | Display name shown by the authenticator during passkey registration |
| `WEBAUTHN_ORIGINS` | No | _(derived from `WEBAUTHN_RP_ID`)_ | Comma-separated list of allowed origins for passkey ceremonies |
| `DEFAULT_SSH_PORT` | No | `22` | Pre-filled SSH port for new devices (per-device override always wins) |
| `DEFAULT_API_PORT` | No | `443` | Pre-filled REST API port for new devices |

**Important:** If you rely on auto-generated secrets, **back up `data/.secrets.json`** alongside the SQLite database. Losing it invalidates all sessions and makes stored device passwords unrecoverable. `ENCRYPTION_KEY` must be exactly 64 hex characters — if changed after devices are added, existing encrypted passwords won't decrypt.

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

| | Community | License 30 | License 50 | License Unlimited |
|---|-----------|-----------|------------|------------------|
| **Devices** | up to 10 | up to 30 | up to 50 | Unlimited |
| **Price** | €0 | €99 (one-time) | €149 (one-time) | €399 (one-time) |
| **Updates** | — | €39/year (optional) | €49/year (optional) | €99/year (optional) |
| **All features** | ✓ | ✓ | ✓ | ✓ |

- **Perpetual license** — the software works forever on the purchased version
- **Update subscription** — grants access to new versions (optional, not required)
- **Self-hosted** — your data stays on your server, offline license activation
- No account required, no telemetry, no usage tracking — none of your fleet/device/config data ever leaves the server
- The only outbound traffic is optional read-only public-feed checks (RouterOS CVE feed from NVD, latest RouterOS version from MikroTik, Manager update from GitHub Releases) — GET-only, carry no data, individually disable-able, and degrade gracefully offline

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
- **Auth**: JWT + bcrypt, role-based (superadmin / admin / operator / viewer) with per-site scoping
- **MikroTik**: SSH (node-ssh) + REST API (https module) + SNMP (net-snmp)
- **Encryption**: AES-256-GCM for stored device passwords
- **Container**: Docker, node:22-alpine (multi-stage build)
