# 🖥️ Dell Latitude 5290 Homelab

> Repurposing a retired Dell Latitude 5290 business laptop into a 24/7 low-power homelab —
> running home automation, self-hosted services, and local AI workloads on constrained hardware.

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Hardware](https://img.shields.io/badge/hardware-Dell%20Latitude%205290-blue)
![OS](https://img.shields.io/badge/OS-CasaOS-blue)
![Home Assistant](https://img.shields.io/badge/Home%20Assistant-container-41BDF5?logo=home-assistant)

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Hardware Specs](#-hardware-specs)
- [System Architecture](#-system-architecture)
- [OS Installation Journey](#-os-installation-journey)
- [Services](#-services)
  - [Home Assistant](#home-assistant)
  - [Reverse Proxy](#reverse-proxy)
  - [Monitoring & Logging](#monitoring--logging)
  - [Local AI Runtime](#local-ai-runtime)
- [Networking](#-networking)
- [Lessons Learned](#-lessons-learned)
- [Roadmap](#️-roadmap)

---

## 🔍 Project Overview

This project documents how I turned a Dell Latitude 5290 — a retired business laptop — into a
dedicated homelab node. Rather than spending money on dedicated server hardware, the goal was to
demonstrate practical self-hosting on constrained, real-world hardware.

**What this covers:**
- OS installation (including the failures) on laptop hardware
- Container-based service deployment and hardening
- Home automation via Home Assistant
- Networking and local DNS configuration
- Early experiments with local AI inference (no cloud, no GPU)

---

## 💻 Hardware Specs

| Component | Details |
|-----------|---------|
| **Model** | Dell Latitude 5290 |
| **CPU** | Intel Core i5 (U-series) — 4C/8T |
| **RAM** | 16 GB DDR4 |
| **Storage** | 512 GB NVMe SSD |
| **Network** | Gigabit Ethernet (USB adapter) + Intel Wi-Fi |
| **Power** | UPS-backed outlet for short outage tolerance |
| **Estimated idle draw** | ~10–15W |

> **Why a laptop?** Built-in battery acts as a mini-UPS. Thermally designed for sustained load.
> Quiet. Low idle power. Easy to source second-hand.

---

## 🏗️ System Architecture

```
                ┌────────────────────────┐
  Internet ─────│  Router / Firewall     │
                └──────────┬─────────────┘
                           │ Gigabit Ethernet
                ┌──────────▼─────────────┐
                │   Dell Latitude 5290   │
                │     (CasaOS Host)      │
                │                        │
                │  ┌──────────────────┐  │
                │  │  Reverse Proxy   │──┼──► Internal HTTPS routing
                │  └────────┬─────────┘  │
                │           │             │
                │  ┌────────▼─────────┐  │
                │  │  Home Assistant  │  │   ← Smart home hub
                │  └──────────────────┘  │
                │  ┌──────────────────┐  │
                │  │   Monitoring     │  │   ← Metrics & logs
                │  └──────────────────┘  │
                │  ┌──────────────────┐  │
                │  │   Local AI       │  │   ← Offline inference
                │  └──────────────────┘  │
                └────────────────────────┘
```

All services run as containers on the host. Host networking is used where local device discovery
(mDNS, multicast) is required.

---

## 🖥️ OS Installation Journey

Getting CasaOS stable on the 5290 took three attempts. This is documented in full because these
are the kinds of failures that don't show up in guides.

> CasaOS installs on top of a base Linux OS (Ubuntu/Debian). The install challenges below were
> hitting the underlying OS first, then layering CasaOS on top once the base was stable.

### Attempt 1 — Direct USB Install ❌

**Steps:**
1. Flashed ISO with Balena Etcher
2. Booted in UEFI mode
3. Installer launched

**Result:**
```
FAILED — Storage controller mismatch during partition stage.
Disk detection was inconsistent; installer couldn't reliably see the NVMe drive.
```

---

### Attempt 2 — BIOS & Storage Mode Tuning ❌

**Changes made:**
1. Updated BIOS to latest version
2. Switched storage controller mode (`RAID → AHCI`)
3. Disabled Secure Boot
4. Recreated installation media

**Result:**
```
FAILED — Installer completed, but bootloader failed after first reboot.
EFI entries were missing/corrupt.
```

---

### Attempt 3 — Clean Repartition + Manual Boot Repair ✅

**Steps:**
1. Booted a live environment
2. Wiped the drive and recreated a fresh GPT partition table
3. Reinstalled base OS (Ubuntu/Debian)
4. Repaired EFI entries manually, then ran the CasaOS installer on top

```bash
# Example: rebuilding EFI entry post-install (Ubuntu base)
efibootmgr --create \
  --disk /dev/nvme0n1 \
  --part 1 \
  --label "Ubuntu" \
  --loader /EFI/ubuntu/grubx64.efi

# Then install CasaOS on top of the stable base:
curl -fsSL https://get.casaos.io | sudo bash
```

**Result:** ✅ System booted and has run stable since.

---

## 🧰 Services

### Home Assistant

Deployed as a container with **host networking** so it can see local devices via mDNS/multicast
without a reflector.

```yaml
# docker-compose.yml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    network_mode: host
    volumes:
      - /opt/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Issues hit and how they were fixed:**

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| mDNS unreliable across VLANs | mDNS doesn't cross VLAN boundaries | Adjusted multicast/avahi settings; moved key devices to static host mappings |
| Zigbee bridge reconnect loops | Intermittent USB reconnect | Moved to static host mapping; added restart guard |
| Slow startup (disk I/O) | Other containers saturating disk on boot | Staggered container start order; added health-check dependencies |

---

### Reverse Proxy

> 📝 **[Fill in: Nginx Proxy Manager / Caddy / Traefik]**

Handles internal HTTPS routing so services are reachable by name rather than port numbers.

```yaml
# Add your reverse proxy compose block here
```

---

### Monitoring & Logging

> 📝 **[Fill in: Netdata / Grafana + Prometheus / Uptime Kuma / etc.]**

Tracks host metrics (CPU, memory, disk, thermals) and container health.

Key things monitored:
- CPU temperature (laptop thermals under sustained load)
- NVMe health / disk I/O
- Container uptime and restart counts
- Network throughput

---

### Local AI Runtime

> 📝 **[Fill in: Ollama / LocalAI / llama.cpp]**

Running local LLM inference without cloud dependency. The 5290 has no discrete GPU, so this
is **CPU-only** inference.

**Models tested:**

> 📝 **[Fill in: e.g., llama3:8b, phi3:mini, mistral:7b]**

**Performance notes:**
- CPU-only limits practical model size to ~7B parameters at Q4 quantisation
- 16 GB RAM is the main constraint
- Better suited for background/batch tasks than interactive chat at this scale

```bash
# Example: running Ollama
# ollama run phi3:mini
```

---

## 🌐 Networking

| Service | Internal Address | Port |
|---------|-----------------|------|
| Home Assistant | `homeassistant.local` | 8123 |
| Reverse Proxy | `proxy.local` | 81 (admin) |
| Monitoring | `monitor.local` | — |
| Local AI API | `ai.local` | — |

> 📝 **[Fill in your actual local DNS / split-DNS setup — Pi-hole, AdGuard, router-level, etc.]**

---

## 📚 Lessons Learned

**1. Storage controller mode is critical for NVMe installs**
On Dell Latitude hardware, the BIOS defaults to RAID mode which many Linux installers can't handle.
Switch to AHCI before attempting install.

**2. Always repair EFI entries manually after a failed boot**
Rather than reinstalling from scratch, `efibootmgr` can often rescue a failed bootloader without
touching the installed OS.

**3. mDNS does not cross VLAN boundaries**
Home Assistant relies on mDNS for device discovery. If you have VLAN segmentation, you either need
an mDNS reflector (`avahi-daemon` with `allow-interfaces`) or static host mappings.

**4. A laptop's battery makes it a better server than you'd expect**
The built-in battery absorbs short power interruptions without needing a UPS — at least for brief
outages. Combined with a UPS-backed outlet, you get decent resilience.

**5. Local AI on CPU is viable but limited**
Small quantised models (phi3:mini, llama3:8b Q4) are usable on CPU-only hardware. Latency is too
high for fast interactive chat but fine for background/automated tasks.

---

## 🗺️ Roadmap

- [ ] Add infrastructure-as-code (Ansible playbook or setup shell scripts) for reproducible install
- [ ] Expand observability: dashboards, alerting, log aggregation
- [ ] Benchmark local AI throughput (tokens/sec per model, RAM usage)
- [ ] Document VLAN/network layout with a proper network diagram
- [ ] Backup strategy (Restic or Borgbackup to NAS or cloud cold storage)
- [ ] Add `docker-compose.yml` files for each service stack

---

## 📁 Repo Structure

```
dell-5290-homelab/
├── README.md               ← You are here
├── docker/
│   ├── homeassistant/
│   │   └── docker-compose.yml
│   ├── proxy/
│   │   └── docker-compose.yml
│   └── monitoring/
│       └── docker-compose.yml
├── scripts/
│   ├── setup.sh            ← Bootstrap script
│   └── backup.sh           ← Backup automation
└── docs/
    ├── os-installation.md  ← Detailed CasaOS install notes
    ├── home-assistant.md   ← HA configuration deep dive
    ├── networking.md       ← Network layout and DNS config
    └── local-ai.md         ← AI runtime setup and benchmarks
```

---

*Hardware is cheap. Documentation is how you prove you know what you're doing.*
