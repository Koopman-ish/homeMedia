# 🏠 Koopman Homelab

A full-featured self-hosted homelab built on Ubuntu Server with Docker Compose. Provides a family media server, cloud storage, photo backup, SSO authentication, AI assistant, automated monitoring, and security — all accessible from anywhere via Tailscale.

---

## 📋 Table of Contents

- [Architecture](#architecture)
- [Stack Overview](#stack-overview)
- [Hardware](#hardware)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Installation](#installation)
- [Service URLs](#service-urls)
- [Environment Variables](#environment-variables)
- [Storage Layout](#storage-layout)
- [Family Access](#family-access)
- [Maintenance](#maintenance)
- [Roadmap](#roadmap)

---

## Architecture

```
Internet
    ↓
Tailscale (remote access, zero-config VPN)
    ↓
Traefik v3 (reverse proxy, HTTPS, mkcert wildcard cert)
    ↓
Authentik (SSO / Identity Provider)
    ↓
┌─────────────────────────────────────────────┐
│              Service Layer                  │
├──────────────┬──────────────┬───────────────┤
│    Media     │     NAS      │ Infrastructure│
│  Jellyfin    │  Nextcloud   │   Grafana     │
│  Jellyseerr  │   Immich     │  Prometheus   │
│  Radarr      │              │    Loki       │
│  Sonarr      │              │   AdGuard     │
│  Lidarr      │              │  Portainer    │
│  Prowlarr    │              │               │
│  Bazarr      │              │               │
│  qBittorrent │              │               │
│  (ProtonVPN) │              │               │
└──────────────┴──────────────┴───────────────┘
    ↓
AdGuard Home (DNS, ad blocking, *.homelab.lan resolution)
    ↓
mergerfs (12TB pooled storage across 3 drives)
```

---

## Stack Overview

### 🔐 Security & Access
| Service | Purpose |
|---|---|
| Traefik v3 | Reverse proxy, HTTPS termination, wildcard cert via mkcert |
| Authentik | SSO / Identity Provider for all services |
| AdGuard Home | Network-wide DNS, ad blocking, `*.homelab.lan` resolution |
| Tailscale | Zero-config remote access from anywhere |
| CrowdSec | Community threat intelligence, IP blocking at Traefik layer |
| Fail2Ban | Brute force protection (SSH, Nextcloud, Traefik) |
| ProtonVPN/Gluetun | VPN kill-switch for all torrent traffic |

### 🎬 Media
| Service | Purpose |
|---|---|
| Jellyfin | Media server — movies, TV, anime, music, books |
| Jellyseerr | Family media request portal |
| Radarr | Movie automation |
| Sonarr | TV series + anime automation |
| Lidarr | Music automation |
| Prowlarr | Central indexer manager |
| Bazarr | Automatic subtitle downloads |
| qBittorrent | Torrent client (routed through ProtonVPN) |
| FlareSolverr | Cloudflare bypass for protected indexers |
| Unpackerr | Auto-extract RAR/ZIP archives |
| Maintainerr | Auto-cleanup: removes unwatched content after 120 days |

### ☁️ NAS / Family
| Service | Purpose |
|---|---|
| Nextcloud | Google Drive replacement — family file storage |
| Immich | Google Photos replacement — photo/video backup |
| Immich ML | Face recognition, smart search (CPU mode) |

### 📊 Monitoring
| Service | Purpose |
|---|---|
| Grafana | Dashboards and visualisation |
| Prometheus | Metrics collection (15s interval, 30d retention) |
| Loki | Log aggregation |
| Promtail | Container log shipping to Loki |
| cAdvisor | Per-container CPU/RAM/network metrics |
| Node Exporter | Host-level metrics |
| Speedtest Exporter | Internet speed history (every 30 mins) |

### 🤖 AI (Stopped — pending RAM upgrade)
| Service | Purpose |
|---|---|
| Hermes | Claude AI agent with web dashboard |
| n8n | Workflow automation |

### 🗂️ Dashboards
| Service | Purpose |
|---|---|
| Homepage | Family landing page |
| Portainer | Container management |
| Watchtower | Automatic container image updates |

---

## Prerequisites

- Ubuntu Server (18.04 or newer)
- Docker + Docker Compose
- Git
- mkcert (for local SSL certificates)
- Tailscale account
- ProtonVPN subscription (WireGuard)
- Anthropic API key (for Hermes)

---

## Directory Structure

```
/opt/containers/
├── .env                          # Global secrets (never commit)
├── networking/
│   ├── docker-compose.yml        # Traefik, AdGuard, Tailscale
│   ├── config/traefik/
│   │   ├── certs/                # mkcert wildcard certificates
│   │   └── dynamic/              # File provider configs (hermes.yml, tls.yml)
│   └── logs/traefik/             # Access logs (for Fail2Ban)
├── security/
│   ├── docker-compose.yml        # Authentik, CrowdSec, Fail2Ban
│   └── config/
│       ├── authentik/
│       ├── crowdsec/
│       └── fail2ban/
├── media/
│   ├── docker-compose.yml        # Full media stack
│   ├── config/                   # Per-service config dirs
│   └── scripts/                  # cold_storage.py etc.
├── nas/
│   ├── docker-compose.yml        # Nextcloud + Immich
│   └── config/
│       ├── nextcloud/
│       ├── immich/
│       └── mkcert-rootCA.crt     # CA cert for container SSL trust
├── monitoring/
│   ├── docker-compose.yml        # Prometheus, Grafana, Loki etc.
│   └── config/
│       ├── prometheus/prometheus.yml
│       ├── loki/loki.yml
│       ├── promtail/promtail.yml
│       └── grafana/provisioning/
├── ai/
│   ├── docker-compose.yml        # Hermes, n8n, Ollama
│   ├── config/hermes/            # Hermes config + SOUL.md
│   ├── config/hermes-josh/       # Josh's personal AI profile
│   ├── config/hermes-family/     # Family AI profile
│   ├── users/                    # Per-user compose files
│   └── hermes-src/               # Hermes source (locally built image)
└── dashboards/
    ├── docker-compose.yml        # Homepage, Portainer, Watchtower
    └── config/homepage/          # services.yaml, settings.yaml etc.

/mnt/
├── disk1/                        # 931GB HDD (direct mount)
├── disk2/                        # 5.5TB HDD (direct mount)
├── disk3/                        # 5.5TB HDD (direct mount)
└── media/                        # mergerfs pool (all 3 drives)
    ├── library/
    │   ├── movies/
    │   ├── tv/
    │   ├── anime/
    │   ├── music/
    │   ├── books/
    │   └── downloads/
    │       ├── complete/
    │       └── incomplete/
    ├── family/
    │   ├── nextcloud/data/
    │   └── immich/
    └── cold-storage/             # Auto-archived content (120 days)
        ├── movies/
        └── tv/
```

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/Koopman-ish/homeMedia.git /opt/containers
cd /opt/containers
```

### 2. Create the environment file

```bash
cp .env.example .env
nano .env
# Fill in all required values
```

### 3. Generate SSL certificates

```bash
# Install mkcert
sudo apt install -y libnss3-tools
curl -Lo mkcert https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64
chmod +x mkcert && sudo mv mkcert /usr/local/bin/

# Create local CA and wildcard cert
mkcert -install
mkdir -p /opt/containers/networking/config/traefik/certs
cd /opt/containers/networking/config/traefik/certs
mkcert "homelab.lan" "*.homelab.lan"
```

### 4. Create Docker networks

```bash
docker network create proxy
docker network create media
docker network create nas
docker network create security
docker network create monitoring
docker network create ai
```

### 5. Set up storage

```bash
# Create storage directories
mkdir -p /mnt/media/library/{movies,tv,anime,music,books}
mkdir -p /mnt/media/library/downloads/{complete,incomplete}
mkdir -p /mnt/media/family/nextcloud/data
mkdir -p /mnt/media/family/immich/{library,thumbs,uploads,profile,backups}
mkdir -p /mnt/media/cold-storage/{movies,tv}
sudo chown -R 1000:1000 /mnt/media/
```

### 6. Start stacks in order

```bash
# Networking first
cd /opt/containers/networking && docker compose up -d

# Security
cd /opt/containers/security && docker compose up -d

# NAS
cd /opt/containers/nas && docker compose up -d

# Media
cd /opt/containers/media && docker compose up -d

# Monitoring
cd /opt/containers/monitoring && docker compose up -d

# Dashboards
cd /opt/containers/dashboards && docker compose up -d
```

### 7. Configure DNS

Set router DNS to:
- Primary: `<<LOCAL_IP>>` (AdGuard)
- Secondary: `1.1.1.1` (fallback)

Add Tailscale split DNS for `homelab.lan` → `<<LOCAL_IP>>` in the Tailscale admin console.

### 8. Install CA cert on client devices

```bash
# Copy rootCA.pem from server
cat ~/.local/share/mkcert/rootCA.pem

# Mac
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain rootCA.pem

# Windows: double-click rootCA.pem → Install → Trusted Root CAs
# iOS/Android: email the .pem file to device, open and trust
```

---

## Service URLs

| Service | URL | Notes |
|---|---|---|
| Homepage | `https://home.homelab.lan` | Family landing page |
| Jellyfin | `https://jellyfin.homelab.lan` | Direct SSO: `/sso/OID/start/Authentik` |
| Jellyseerr | `https://requests.homelab.lan` | Media requests |
| Nextcloud | `https://nextcloud.homelab.lan` | Family files |
| Immich | `https://immich.homelab.lan` | Family photos |
| Authentik | `https://authentik.homelab.lan` | SSO admin |
| AdGuard | `https://adguard.homelab.lan` | DNS admin |
| Grafana | `https://grafana.homelab.lan` | Monitoring |
| Portainer | `https://portainer.homelab.lan` | Container management |
| Traefik | `https://traefik.homelab.lan` | Proxy dashboard |
| Radarr | `https://radarr.homelab.lan` | Movies |
| Sonarr | `https://sonarr.homelab.lan` | TV/Anime |
| Lidarr | `https://lidarr.homelab.lan` | Music |
| Prowlarr | `https://prowlarr.homelab.lan` | Indexers |
| Bazarr | `https://bazarr.homelab.lan` | Subtitles |
| qBittorrent | `http://10.0.0.32:8080` | Torrent client (no Traefik) |
| Hermes | `https://hermes.homelab.lan` | AI agent |
| n8n | `https://n8n.homelab.lan` | Automation |

---

## Environment Variables

Copy `.env.example` to `.env` and fill in all values. **Never commit `.env` to git.**

| Variable | Description |
|---|---|
| `TZ` | Timezone e.g. `Africa/Johannesburg` |
| `PUID` / `PGID` | User/Group ID (run `id` to get) |
| `DOMAIN` | Local domain e.g. `homelab.lan` |
| `CONFIG_ROOT` | Config base path e.g. `/opt/containers` |
| `MEDIA_ROOT` | Media storage path e.g. `/mnt/media` |
| `AUTHENTIK_SECRET_KEY` | Random 32-byte hex key |
| `AUTHENTIK_POSTGRES_PASSWORD` | Authentik DB password |
| `NEXTCLOUD_POSTGRES_PASSWORD` | Nextcloud DB password |
| `NEXTCLOUD_ADMIN_USER` | Nextcloud admin username |
| `NEXTCLOUD_ADMIN_PASSWORD` | Nextcloud admin password |
| `IMMICH_POSTGRES_PASSWORD` | Immich DB password |
| `GRAFANA_ADMIN_USER` | Grafana admin username |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password |
| `VPN_PROVIDER` | VPN provider e.g. `protonvpn` |
| `VPN_WIREGUARD_PRIVATE_KEY` | WireGuard private key from VPN provider |
| `VPN_SERVER_COUNTRY` | VPN server country e.g. `Netherlands` |
| `CROWDSEC_BOUNCER_API_KEY` | Generated by CrowdSec after first run |
| `ANTHROPIC_API_KEY` | Anthropic API key for Hermes |
| `N8N_LICENSE_ACTIVATION_KEY` | n8n license key |

---

## Storage Layout

Three physical drives pooled via **mergerfs** into `/mnt/media`:

```
/dev/sda1  (931GB)  → /mnt/disk1
/dev/sdb1  (5.5TB)  → /mnt/disk2
/dev/sdd1  (5.5TB)  → /mnt/disk3
                         ↓ mergerfs
                    /mnt/media (12TB)
```

fstab entry:
```
/mnt/disk1:/mnt/disk2:/mnt/disk3  /mnt/media  fuse.mergerfs  defaults,allow_other,use_ino,category.create=mfs  0  0
```

> ⚠️ No RAID — if a drive fails, only data on that drive is lost. SnapRAID parity protection planned.

---

## Family Access

### User Accounts
All family accounts created in Authentik (`Directory → Users`).
Format: `firstname.lastname` — assigned to `family` group.

| Username | Role |
|---|---|
| `josh.koopman` | Admin — full access |
| `User2` | Family member |
| `User3` | Family member |
| `User4` | Family member |
| `User5` | Family member |
| `User6` | Family member |

### SSO Flow
Single sign-on via Authentik covers: Nextcloud, Immich, Jellyfin, Hermes, Homepage.
Log in once at `https://authentik.homelab.lan` — all services accessible without re-entering credentials.

### Mobile Setup
1. Install **Tailscale** — connect to homelab from anywhere
2. Install **Jellyfin** app — server: `http://<<LOCAL_IP>>:8096`
3. Install **Immich** app — server: `https://immich.homelab.lan`, enable WiFi-only auto-backup
4. Install mkcert CA certificate (no SSL warnings)

---

## Maintenance

### Cold Storage (auto)
Movies/TV not watched in 120 days are automatically moved to cold storage.
Files removed but metadata/thumbnails remain in Jellyfin.
Re-request via Jellyseerr to restore.

Script: `/opt/containers/media/scripts/cold_storage.py`
Cron: Every Sunday at 3 AM.

### Container Updates
Watchtower runs at 4 AM daily and updates containers automatically.
Rolling restarts — no downtime.

### Backups
> ⚠️ Offsite backup not yet configured. Critical data (Nextcloud, Immich) should be backed up to Backblaze B2 or similar.

---
### 🏆 Recommended Build
*Proper NAS hardware, handles everything comfortably*

| Component | Recommendation | Price (ZAR) | Why |
|---|---|---|---|
| **CPU** | AMD Ryzen 7 5700G | R3,500–5,000 | 8 cores, integrated Vega GPU (QuickSync for Jellyfin), low TDP |
| **Motherboard** | ASUS Prime B550M-A | R2,500–3,500 | 6× SATA, 2× M.2, solid VRM for 24/7 |
| **RAM** | 64GB DDR4 ECC (4× 16GB) | R4,000–6,000 | ECC protects data integrity, headroom for AI models |
| **GPU** | NVIDIA RTX 3060 12GB | R7,000–9,000 | Hardware transcoding + Immich ML + Ollama (Qwen 7B+) |
| **OS Drive** | Samsung 980 Pro 500GB NVMe | R1,000–1,500 | Fast container startup, system responsiveness |
| **Media Drives** | 2× Seagate IronWolf Pro 16TB | R5,000–6,500 each | NAS-rated, CMR (not SMR), 3-year warranty, built for 24/7 |
| **Case** | Fractal Design Node 804 | R2,500–3,500 | 8× hot-swap bays, excellent airflow, compact mATX |
| **PSU** | Corsair RM750x 750W | R2,000–2,500 | Modular, 80+ Gold, reliable for 24/7 |
| **UPS** | APC Smart-UPS 1500VA | R4,000–6,000 | Pure sine wave, USB management, runtime management |
| **Network** | Intel X550-T1 10GbE card | R1,500–2,500 | Eliminates 1GbE bottleneck for file transfers + 4K streaming |

**Total (excl. existing drives):** ~R35,000–50,000

**Expected improvement:** 4K HDR hardware transcoding for 4 simultaneous streams, local AI models (Qwen 7B, Llama 3), handles 6 family members without breaking a sweat.
---

## Contributing

This is a personal homelab project. Feel free to fork and adapt for your own use.

---

*Built with 🛠️ by Josh Koopman — Cape Town, South Africa*
