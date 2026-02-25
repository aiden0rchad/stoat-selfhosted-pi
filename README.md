# Stoat Chat — Orange Pi 3B Self-Hosting Guide 🦦

> A complete guide to deploying a private, self-hosted Discord alternative on an Orange Pi 3B with NVMe storage. Written for intermediate-to-advanced homelabbers.

---

> [!WARNING]
> ## ⚠️ Security Warning — READ THIS FIRST
>
> You are about to expose a server to the internet. If done incorrectly, a misconfigured server can give bad actors a foothold into **your entire home network**. Please understand the risks before proceeding.
>
> **Minimum recommended knowledge:**
> - Basic Linux comfort (SSH, `apt`, `systemctl`, file editing)
> - Understanding of VLANs, subnets, and firewall rules
> - Familiarity with your router/switch admin interface
>
> **Strongly recommended:**
> - Place the Orange Pi in a **DMZ or isolated VLAN** — physically or logically cut off from your main LAN and any IoT devices
> - Use **Cloudflare Tunnel** instead of open port forwarding — this removes the need to expose your home IP at all
> - Do NOT run this on your main desktop or home server if you can't isolate it
>
> If you're not sure what a DMZ is or how to set up a VLAN, this guide will explain the concept — but please do additional research before exposing this to the internet.

---

## Hardware Used

| Component | Detail |
|---|---|
| **Board** | Orange Pi 3B (RK3566, 8GB LPDDR4) |
| **Storage** | M.2 NVMe SSD (PCIe 2.0, any size — 128GB+ recommended) |
| **Network** | Ethernet (wired, not WiFi) |
| **Power** | 5V/4A USB-C supply |
| **Cooling** | Passive heatsink or small fan recommended |
| **OS** | Armbian Minimal/IOT (Debian Bookworm, Kernel 6.18+) |

---

## Part 1: Hardware Setup

### 1.1 — Install the NVMe SSD

1. Power off the board completely
2. Insert your M.2 NVMe into the slot on the underside of the Orange Pi 3B
3. Secure it with the standoff screw
4. Confirm it's seated fully — a half-inserted drive will not be detected

### 1.2 — Flash the OS to a MicroSD Card (Temporary Boot)

You'll boot from SD first to set up the NVMe, then move everything to NVMe.

> [!WARNING]
> **Do NOT use the official Orange Pi images.** They are distributed via Google Drive with no proper verification mirror, ship outdated kernels with unaudited vendor patches, and route all package updates through Chinese CDNs (Huawei mirrors) that are baked into the OS. There are no independent security audits of these images. For a server you're exposing to the internet, this is an unacceptable risk.

**Use Armbian Minimal/IOT instead:**

1. Go to **[armbian.com/orange-pi-3b](https://www.armbian.com/orange-pi-3b/)** and download the **Minimal / IOT** image
   - `~322MB`, much leaner than a desktop image
   - Ships with a recent mainline kernel (6.18+)
   - Distributed via Armbian's own verified mirrors — no Google Drive
2. **Verify the download before flashing** — click the SHA hash and PGP signature links on the download page:

   **Linux / macOS:**
   ```bash
   sha256sum Armbian_*.img.xz
   # or on macOS:
   shasum -a 256 Armbian_*.img.xz
   ```
   **Windows (PowerShell):**
   ```powershell
   Get-FileHash Armbian_*.img.xz -Algorithm SHA256
   ```
   Compare the output to the SHA value on the Armbian download page — must match exactly.
3. Flash to a microSD card using [Balena Etcher](https://etcher.balena.io/)
4. Insert the SD card, connect Ethernet, and power on
5. First boot takes ~2 minutes. Default credentials: `root` / `1234` (you'll be forced to change this on first login)

**Why Armbian?**
- ✅ Open source build system — every image is built reproducibly in public CI
- ✅ Standard Debian/Ubuntu package repos — no vendor CDN baked in
- ✅ GPG-signed images you can verify before flashing
- ✅ Mainline kernel with proper upstream security patches
- ✅ Active security team with rapid CVE response
- ✅ No pre-installed vendor bloat or mystery services

### 1.3 — Initial System Hardening

**Change the default password immediately:**
```bash
passwd
```

**Create a non-root user:**
```bash
sudo adduser yourusername
sudo usermod -aG sudo yourusername
```

**Disable root SSH login:**
```bash
sudo nano /etc/ssh/sshd_config
# Set: PermitRootLogin no
# Set: PasswordAuthentication yes  (until you set up SSH keys)
sudo systemctl restart sshd
```

**Set up SSH key authentication (from your workstation):**

**Linux / macOS:**
```bash
ssh-keygen -t ed25519 -C "orangepi"
ssh-copy-id yourusername@<orange-pi-ip>
```

**Windows (PowerShell / Windows Terminal):**
```powershell
ssh-keygen -t ed25519 -C "orangepi"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh yourusername@<orange-pi-ip> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Then on the Pi, disable password auth:
```bash
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart sshd
```

**Update the system:**
```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

---

## Part 2: NVMe Setup — Move Everything Off the SD Card

> [!TIP]
> Running the OS and all Docker data from NVMe gives you dramatically better performance and longevity — SD cards degrade rapidly under database write loads.

### 2.1 — Identify the NVMe Drive

```bash
lsblk
```

You should see something like:
```
mmcblk0   (SD card — your current root)
nvme0n1   (NVMe — empty)
```

### 2.2 — Partition and Format the NVMe

```bash
sudo fdisk /dev/nvme0n1
# Press: n (new partition), p (primary), 1 (partition 1)
# Accept defaults for start/end (uses full disk)
# Press: w (write and exit)

sudo mkfs.ext4 /dev/nvme0n1p1
```

### 2.3 — Clone the SD Card to NVMe

```bash
sudo apt install -y rsync

sudo mkdir /mnt/nvme
sudo mount /dev/nvme0n1p1 /mnt/nvme

sudo rsync -axHAWXS --numeric-ids --info=progress2 / /mnt/nvme
```

### 2.4 — Update Boot Configuration to Use NVMe

Get the NVMe partition UUID:
```bash
sudo blkid /dev/nvme0n1p1
# Note the UUID value
```

Edit `/mnt/nvme/etc/fstab`:
```bash
sudo nano /mnt/nvme/etc/fstab
# Replace the root entry UUID with your NVMe UUID
# Should look like: UUID=<nvme-uuid>  /  ext4  defaults  0  1
```

Update the bootloader:
```bash
# Orange Pi 3B uses U-Boot with extlinux
sudo mkdir -p /mnt/nvme/boot/extlinux
sudo nano /mnt/nvme/boot/extlinux/extlinux.conf
# Change root= parameter to: root=/dev/nvme0n1p1
```

### 2.5 — Reboot to NVMe

```bash
sudo reboot
```

After reboot, verify you're running from NVMe:
```bash
df -h /
# Should show /dev/nvme0n1p1 as root
```

Once confirmed, you can remove the SD card (or keep it as emergency fallback).

---

## Part 3: Docker Installation

### 3.1 — Install Docker Engine

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 3.2 — Configure Docker Logging (Prevent Log Bloat)

```bash
sudo nano /etc/docker/daemon.json
```

Paste:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

### 3.3 — Set Up Swap (Recommended for 4GB boards; optional for 8GB)

If you ever do memory-intensive operations (like large image builds):
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Part 4: Deploy Stoat Chat

### 4.1 — Clone This Repository

```bash
git clone https://github.com/aiden0rchad/stoat-selfhosted-pi.git stoat
cd stoat
```

### 4.2 — Configure Environment

```bash
cp .env.web.example .env
nano .env
```

Key settings to change:
- `DOMAIN` — your domain name or local IP
- `MONGODB_PASSWORD` — set a strong password
- `RABBIT_PASSWORD` — set a strong password

### 4.3 — Generate Vapid Keys

```bash
docker run --rm -it node:20-alpine npx web-push generate-vapid-keys
# Copy the public and private keys into your .env
```

### 4.4 — Start the Stack

```bash
docker compose pull       # Pull all pre-built ARM64 images (~3-5GB)
docker compose up -d      # Start everything in the background
```

First boot takes 2–5 minutes while databases initialize. Check status:
```bash
docker compose ps
docker compose logs -f    # Ctrl+C to stop following
```

### 4.5 — Access Locally

- **Stoat Chat web app**: `http://<orange-pi-ip>:13080`
- **Admin Panel**: `http://<orange-pi-ip>:13000`
- **Create your admin account** via the web app first, then promote it in the admin panel

---

## Part 5: Network Security — Isolation & Exposure

> [!DANGER]
> **This is the most critical section.** Skip it and you risk your entire home network.

### 5.1 — Why You Need Isolation

When you host a public-facing server, anyone who finds it can probe it for vulnerabilities. If that server is on your main home network (same subnet as your laptops, NAS, cameras), a compromised server gives an attacker access to everything else.

### Option A: DMZ (Recommended for Router/Switch Users)

A **DMZ** (Demilitarized Zone) is a separate network segment isolated from your LAN by firewall rules. The Orange Pi lives there — it can reach the internet, but it **cannot** reach your main LAN devices.

**Typical firewall rules:**
```
DMZ → WAN:  ALLOW (internet access)
DMZ → LAN:  BLOCK (cannot touch your home devices)
LAN → DMZ:  ALLOW from your IP only (for administration)
WAN → DMZ:  ALLOW specific ports only (80, 443, UDP 50000-50100 for voice)
```

Set this up in your router's firewall or a dedicated firewall appliance (pfSense, OPNsense, UniFi).

### Option B: Cloudflare Tunnel (Easiest — Hides Your Home IP)

Cloudflare Tunnel creates an **outbound-only** encrypted connection from your server to Cloudflare's network. No ports need to be opened on your router. Your home IP is never exposed.

```bash
# Install cloudflared on the Orange Pi
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloudflare-main.gpg
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# Authenticate with your Cloudflare account
cloudflared tunnel login

# Create a tunnel
cloudflared tunnel create stoat

# Configure it to point at Caddy (the internal reverse proxy)
nano ~/.cloudflared/config.yml
```

`~/.cloudflared/config.yml`:
```yaml
tunnel: <your-tunnel-id>
credentials-file: /home/yourusername/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: chat.yourdomain.com
    service: http://localhost:13080
  - service: http_status:404
```

```bash
# Run as a system service
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

> [!NOTE]
> **Cloudflare Tunnel does NOT support UDP.** This means voice/video (LiveKit WebRTC) will not work through the tunnel. For voice channels, you still need UDP ports 50000–50100 open from the internet to the Orange Pi.
>
> **Workaround:** Use a separate subdomain pointing to your public IP for LiveKit only, while tunneling the main web app.

### Option C: VPN-Only Access (Most Secure — No Public Exposure)

If you only want access for you and your friends (who can all install WireGuard or Tailscale), the most secure option is to **never expose Stoat to the public internet at all**.

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Everyone in your friend group installs Tailscale and joins your tailnet. Stoat is only accessible via the private Tailscale IP. Zero public exposure.

---

## Part 6: SSL/TLS Certificates

If using Cloudflare Tunnel, Cloudflare handles HTTPS for you.

If exposing directly, use Caddy's built-in automatic HTTPS (already included in the compose stack):

```bash
# In your Caddyfile, replace the local IP with your domain:
nano Caddyfile
# Change: :80 to: chat.yourdomain.com
```

Caddy will automatically fetch and renew a Let's Encrypt certificate.

---

## Part 7: Maintenance

### Check Resource Usage
```bash
docker stats
```

### Update the Stack
```bash
docker compose pull
docker compose up -d
```

### Backup Your Data
```bash
# MongoDB backup
docker exec stoat-database-1 mongodump --out /tmp/backup
docker cp stoat-database-1:/tmp/backup ./backup-$(date +%Y%m%d)

# Or just backup the entire data directory:
sudo tar -czf stoat-backup-$(date +%Y%m%d).tar.gz ./data/
```

### View Logs
```bash
docker compose logs api          # API server logs
docker compose logs events       # WebSocket logs
docker compose logs database     # MongoDB logs
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Container won't start | `docker compose logs <service>` for details |
| Can't connect from other devices | Check firewall: `sudo ufw status` |
| Voice/video not working | Ensure UDP 50000–50100 is open to the Pi |
| Slow first startup | Normal — MongoDB initializing (wait 3–5 min) |
| Out of disk space | `docker system prune` to remove old images |
| NVMe not detected | Reseat the M.2 card, check the slot screw |

---

## Hardware Summary

| Spec | Detail |
|---|---|
| **Board** | Orange Pi 3B |
| **CPU** | RK3566 Quad-Core ARM64 @ 1.8GHz |
| **RAM** | 8GB LPDDR4 |
| **Storage** | M.2 NVMe SSD |
| **Network** | Gigabit Ethernet |
| **OS** | Armbian Minimal/IOT (Debian Bookworm, Kernel 6.18+) |
| **Idle Power** | ~3–5W |

---

## What's Optimized in This Build

| Optimization | Detail |
|---|---|
| No memory caps | All 8GB available to services |
| No CPU limits | Full quad-core usage |
| ARM64 targeting | `platform: linux/arm64` on every service |
| KeyDB tuned | 256MB cache + 2 server threads |
| Full LiveKit UDP range | 50000–50100 for best voice/video quality |
| NVMe storage | All data volumes write directly to NVMe |
| 100MB file uploads | Generous limits vs. upstream defaults |

---

*This guide is maintained by the community. If something is out of date or broken, please open an issue or PR.*
