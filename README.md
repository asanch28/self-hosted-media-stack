# Self-Hosted Media Stack (UGREEN NAS + Docker + VPN + Discord Bot)

A fully automated, secure, self-hosted media server running on a **UGREEN NAS** using **Docker**, featuring:

- **Plex Media Server**  
- **Overseerr** (Modern media request portal)  
- **Sonarr** (TV automation)  
- **Radarr** (Movie automation)  
- **Prowlarr** (Indexer management)  
- **qBittorrent** (VPN-routed downloader)  
- **Gluetun** (AirVPN / WireGuard / OpenVPN container)  
- **FlareSolverr** (Bypasses Cloudflare protection for indexers)  
- **Doplarr** (Discord bot for media requests)

This repository documents the **entire deployment**, including networking, VPN routing, bot integration, and best practices for a professional-grade homelab project.

---

## 🚀 Features

### ✅ Automated Media Pipeline  
Sonarr/Radarr automatically fetch content → Prowlarr handles indexers → qBittorrent downloads via VPN → Plex imports instantly.

### ✅ Modern Request UI  
**Overseerr** allows users to request media from a beautiful web dashboard.

### ✅ Cloudflare Bypass for Indexers  
**FlareSolverr** allows Prowlarr/Sonarr/Radarr to access Cloudflare-protected sites.

### ✅ VPN-Routed Downloading  
qBittorrent is forced through **Gluetun** to guarantee privacy.

### ✅ Discord Media Request Bot  
**Doplarr** lets users request movies/TV directly from Discord.

### ✅ Hardware Transcoding (optional)  
Plex supports `/dev/dri` mapping if enabled on the NAS.

```

## 📁 Project Structure

```plaintext
arr-apps/
│
├── docker-compose.yml
├── gluetun/                 
├── qbittorrent/             
├── radarr/                  
├── sonarr/                  
├── prowlarr/                
├── overseerr/               
├── flaresolverr/            
├── plex/                    
└── discord-bot/   

```

## 📦 Services Included
✨ Overseerr

A polished media request dashboard.
Beautiful interface
OAuth login
Requests automatically forwarded to Sonarr/Radarr
Perfect for household or friends sharing your Plex server

✨ FlareSolverr
A headless browser used by Prowlarr to bypass Cloudflare “Access Denied” pages.
Recommended if you're using public indexers such as:

1337x
Nyaa
RARBG clones
TorrentGalaxy
Any site with Cloudflare-JS challenges

## 🏗️ Deployment Steps
PUID=1001
PGID=100
TZ=America/Los_Angeles

DOCKERCONFDIR=/volume1/docker/appdata
DOCKERSTORAGEDIR=/volume1/arr-data

VPN_USER=your_nordvpn_username
VPN_PASS=your_nordvpn_password


## 🧰 Complete Docker Compose (Including Overseerr + FlareSolverr)
version: "3.2"
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=openvpn
      - OPENVPN_CUSTOM_CONFIG=/gluetun/airvpn.ovpn
      - OPENVPN_CUSTOM_CONFIG_OVERRIDE_REMOTE=1
      - AIRVPN_PORT_FORWARDING=on
      - FIREWALL_VPN_INPUT_PORTS=18598
      - TZ=America/Los_Angeles
    volumes:
      - /volume1/docker/qbittorrent-gluetun/gluetun:/gluetun
    ports:
      - "8090:8080"
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: service:gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - /volume1/docker/qbittorrent-gluetun/qbittorrent:/config
      - /volume1/arr-data/torrents:/data/torrents
    depends_on:
      gluetun:
        condition: service_started
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    restart: unless-stopped
    environment:
      - TZ=America/Los_Angeles
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none

  radarr:
    image: ghcr.io/hotio/radarr:latest
    container_name: radarr
    ports: ["7878:7878"]
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/radarr:/config
      - ${DOCKERSTORAGEDIR}:/data
    restart: unless-stopped

  sonarr:
    image: ghcr.io/hotio/sonarr:release
    container_name: sonarr
    ports: ["8989:8989"]
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/sonarr:/config
      - ${DOCKERSTORAGEDIR}:/data
    restart: unless-stopped

  prowlarr:
    image: ghcr.io/hotio/prowlarr:testing
    container_name: prowlarr
    ports: ["9696:9696"]
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/prowlarr:/config
    restart: unless-stopped

  ---
version: '3'

services:
  overseerr:
    image: sctx/overseerr:latest
    container_name: overseerr
    environment:
      #- LOG_LEVEL=debug
      - TZ=America/Los_Angeles
      - PORT=5055 #optional
    ports:
      - 5055:5055
    volumes:
      - ./config:/app/config
    restart: unless-stopped

  plex:
    container_name: plex
    image: ghcr.io/hotio/plex
    ports: ["32400:32400"]
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - PLEX_CLAIM=${PLEX_CLAIM}
      - PLEX_ADVERTISE_URL=http://YOUR_NAS_IP:32400
    volumes:
      - ${DOCKERCONFDIR}/plex:/config
      - ${DOCKERSTORAGEDIR}/media:/data/media
    restart: unless-stopped

  doplarr:
    image: lscr.io/linuxserver/doplarr:latest
    container_name: doplarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - DISCORD__TOKEN=${DISCORD_TOKEN}
      - DISCORD__GUILD_ID=${DISCORD_GUILD_ID}
      - DISCORD__ROLE_ID=${DISCORD_ROLE_ID}
      - DISCORD__REQUEST_CHANNEL_ID=${DISCORD_CHANNEL_ID}
      - RADARR__URL=http://192.168.50.38:7878
      - RADARR__API=<radarr_api_key>
      - SONARR__URL=http://192.168.50.38:8989
      - SONARR__API=<sonarr_api_key>
    restart: unless-stopped
    
## 🧩 Prowlarr + FlareSolverr Integration
## 🌐 Overseerr Setup
## 📡 Discord Automation (Doplarr)
## 🏁 Final Result (Screenshots)

Below are screenshots of the final fully automated media stack running on my UGREEN NAS.

### 📺 Plex Dashboard  
A clean library view with instant metadata refresh powered by Radarr/Sonarr automation.  
![Plex Dashboard](screenshots/plex-dashboard.png)

### 🎬 Overseerr – Media Request Portal  
Users request movies/TV through a modern web UI.  
![Overseerr](screenshots/overseerr.png)

### 🔐 qBittorrent Routed Through Gluetun VPN  
All torrent traffic forced through NordVPN (OpenVPN).  
![qBittorrent](screenshots/qbittorrent.png)

### 🌐 Prowlarr + FlareSolverr (Cloudflare Bypass)  
Public indexers load without Cloudflare errors thanks to FlareSolverr.  
![Prowlarr + Flaresolverr](screenshots/prowlarr-flaresolverr.png)

### 🐳 UGREEN NAS Docker UI  
The entire stack visible and managed through UGREEN’s Docker interface.  
![UGREEN Docker UI](screenshots/ugreen-docker.png)

### 🤖 Discord Media Request Bot (Doplarr)  
Users can request content directly in Discord.  
![Doplarr Bot](screenshots/doplarr.png)

