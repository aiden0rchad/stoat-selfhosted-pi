# Stoat Chat — Orange Pi 3B Edition �

A speed-optimized self-hosted deployment of [Stoat Chat](https://github.com/stoatchat/self-hosted), tuned for the Orange Pi 3B 8GB running headless on NVMe.

## Hardware Profile

| Spec | Detail |
|---|---|
| **Board** | Orange Pi 3B |
| **CPU** | RK3566 Quad-Core ARM64 |
| **RAM** | 8GB LPDDR4 |
| **Storage** | M.2 NVMe SSD (OS + Docker) |
| **Network** | DMZ segment, Cloudflare Tunnel |
| **OS** | Headless Debian/Ubuntu (no GUI) |

## What's Optimized?

| Optimization | Detail |
|---|---|
| **No memory caps** | All 8GB available — services use what they need |
| **No CPU limits** | Full quad-core for all containers |
| **ARM64 targeting** | `platform: linux/arm64` on every service |
| **KeyDB tuned** | 256MB cache + 2 server threads for fast lookups |
| **Admin panel** | 512MB Node.js heap for snappy dashboard |
| **Full LiveKit range** | UDP 50000–50100 for best voice/video quality |
| **NVMe storage** | All data volumes write directly to NVMe |
| **Custom web image** | Nginx Alpine serving your modified frontend |

## Prerequisites

- **Orange Pi 3B** with 8GB RAM
- **Debian or Ubuntu** headless (no desktop environment)
- **M.2 NVMe SSD** installed and formatted (OS + Docker running from it)
- **Docker + Docker Compose** installed

## Quick Setup

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in
```

### 2. Clone This Repo
```bash
git clone --recurse-submodules https://github.com/aiden0rchad/stoat-selfhosted-pi.git stoat
cd stoat
```

### 3. Configure Environment
```bash
cp .env.web.example .env.web
# Edit .env.web if you have a custom domain
```

### 4. Build & Start
```bash
docker compose build   # ~15-20 min first time on ARM
docker compose up -d
```

### 5. Access
- **Stoat Chat**: `http://<orange-pi-ip>:13080`
- **Admin Panel**: `http://<orange-pi-ip>:13000`

## DMZ & Cloudflare Tunnel Notes

Since this box lives in a DMZ:
- **No ports need to be forwarded on your main router** if you use Cloudflare Tunnel
- Add the Cloudflare Tunnel container later pointing to `caddy:80` (the internal Caddy port)
- LiveKit WebRTC **requires UDP ports 50000–50100** — Cloudflare Tunnel does NOT support UDP, so for voice/video you'll still need those ports open from the DMZ to the internet
- Consider running the Cloudflare Tunnel as an additional service in this compose stack when ready

## Performance Tips

- **NVMe = no bottleneck** — MongoDB, MinIO, and all data volumes will be fast
- **Monitor** with `docker stats` to see real-time container resource usage
- **First boot is slow** — images must be pulled (~3-5GB), subsequent starts are instant
- **Keep Docker logging in check**: add the following to `/etc/docker/daemon.json` to prevent logs from filling your SSD:
  ```json
  {
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "10m",
      "max-file": "3"
    }
  }
  ```
