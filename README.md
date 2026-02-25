# Stoat Chat — Raspberry Pi 4 Edition 🍓

A memory-optimized self-hosted deployment of [Stoat Chat](https://github.com/stoatchat/self-hosted), tuned for Raspberry Pi 4 (4GB RAM).

## What's Different from the Standard Version?

| Optimization | Detail |
|---|---|
| **Memory Limits** | Every Docker service has a `mem_limit` cap (~2.5GB total budget) |
| **CPU Pinning** | `cpus` constraints prevent any single service from starving others |
| **ARM64 Targeting** | `platform: linux/arm64` on all services for clean native pulls |
| **Reduced UDP Ports** | LiveKit uses `50000-50020` instead of `50000-50100` |
| **MongoDB Tuning** | `MALLOC_ARENA_MAX=2` reduces glibc memory overhead |
| **KeyDB Tuning** | `--maxmemory 48mb --maxmemory-policy allkeys-lru` |
| **Custom Web Image** | Nginx Alpine serving static frontend (~20MB vs upstream) |
| **Admin Panel Tuning** | `NODE_OPTIONS=--max-old-space-size=128` + cache cleanup |

## Prerequisites

- **Raspberry Pi 4** with **4GB RAM** (8GB strongly recommended if budget allows)
- **Raspberry Pi OS 64-bit** (Bookworm or later)
- **USB 3.0 SSD** — do NOT use a microSD card for the database (MongoDB writes will kill it)
- **Docker + Docker Compose** installed

## Quick Setup

### 1. Enable Swap (Critical for 4GB Pi)
```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

### 2. Clone This Repo
```bash
git clone --recurse-submodules https://github.com/aiden0rchad/stoat-selfhosted-pi.git stoat
cd stoat
```

### 3. Configure Environment
```bash
cp .env.web.example .env.web
```

### 4. Build & Start
```bash
docker compose build   # This will take ~15-30 min on Pi 4
docker compose up -d
```

### 5. Access
- **Stoat Chat**: `http://<pi-ip>:13080`
- **Admin Panel**: `http://<pi-ip>:13000`

## Memory Budget Breakdown

| Service | Limit | Notes |
|---|---|---|
| MongoDB | 512MB | Main database |
| MinIO | 256MB | File storage |
| API | 256MB | Backend API |
| LiveKit | 256MB | Voice/video relay |
| RabbitMQ | 192MB | Message broker |
| Admin Panel | 192MB | Dashboard |
| Events + Autumn + Voice Ingress | 128MB each | WebSocket, files, voice |
| January | 96MB | Link proxy |
| Redis + Caddy + GifBox + Crond + PushD + Web | 64MB each | Light services |
| **Total** | **~2.5GB** | Leaves ~1.5GB for OS + swap |

## Tips

- **First boot is slow** — MongoDB needs to initialize, and images need to be pulled (~3-5GB total)
- **Monitor memory** with `docker stats` to see real-time container usage
- **If OOM occurs**, consider disabling `gifbox` and `pushd` (optional services)
- **Port forwarding**: Open UDP `50000-50020` on your router for voice/video calls
